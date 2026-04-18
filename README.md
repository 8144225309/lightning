# Core Lightning — bLIP-56 (Pluggable Channel Factories)

This is a fork of [Core Lightning](https://github.com/ElementsProject/lightning) implementing [bLIP-56](https://github.com/lightning/blips/pull/56) — the pluggable channel factory protocol described in the [delving bitcoin post](https://delvingbitcoin.org/t/pluggable-channel-factories/1252). Plugins like [superscalar-cln](https://github.com/8144225309/superscalar-cln) handle all factory logic (MuSig2, DW trees, ceremonies); this fork provides the channel-management plumbing that ties factory state to Lightning channel state.

Based on **CLN v25.12**. Changes on the [`blip-56`](https://github.com/8144225309/lightning/tree/blip-56) branch.

## bLIP-56 Wire Protocol

- **Feature bit 270/271** (`pluggable_channel_factories`) — advertised in `init` and `node_announcement` for peer discovery of factory-capable nodes.
- **TLV 65600** (`channel_in_factory`) on `open_channel` and `accept_channel` — carries `factory_protocol_id` (32 bytes), `factory_instance_id` (32 bytes), and `factory_early_warning_time` (u16). Signals that this channel lives inside a factory.
- **Factory plugin-to-plugin messages** use **ODD custommsg** (type 33001) — no new BOLT peer wire message types needed. Factory ceremony traffic (MuSig2 nonces, partial sigs, etc.) is entirely plugin-to-plugin.

## What This Fork Changes

### Channel Opening

- **TLV 65600** on `open_channel`/`accept_channel` — marks channels as factory-hosted. Triggers zero-conf (`minimum_depth=0`) and skips on-chain funding watch.
- **`fundchannel_start` factory params** — `factory_protocol_id`, `factory_instance_id`, `factory_early_warning_time` RPC params populate TLV 65600.
- **`fundchannel_complete` override** — `factory_funding_txid` + `factory_funding_outnum` params specify the DW tree leaf outpoint as channel funding.
- **Fundee zero-conf** — plugin's `openchannel` hook returns `mindepth=0` for factory peers; openingd validates TLV 65600 echo.

### Factory State Changes (Splice-Like)

Factory rotation uses the splice-equivalent flow from the [delving post](https://delvingbitcoin.org/t/pluggable-channel-factories/1252):

1. **STFU quiescence** — channeld enters `stfu` before factory-change, pausing HTLC updates
2. **Batch `commitment_signed`** — channeld signs commitments against BOTH old and new funding outpoints simultaneously, using the `splice_info` TLV (same mechanism as splice)
3. **Multi-outpoint validity** — both outpoints are valid until factory protocol settles
4. **`factory_change_locked`** — plugin signals old state invalidated; channeld calls `channel_update_funding()` to finalize, exits STFU

- **`factory-change` RPC** — triggers the STFU + batch commit flow. Internal channeld wires (7232/7233/7235) coordinate.
- **`factory-forget-channel` RPC** — plugin can drop a channel without commitment broadcast (for cooperative factory close or penalty).

### Utilities

- **`checkutxo` RPC** — UTXO status query for breach detection.

### Bug Fix

- **Wallet crash fix** — `db_cols_account` graceful fallback for missing channels in coin movements.

## Building

```bash
git clone --branch blip-56 https://github.com/8144225309/lightning.git
cd lightning

# Enable MuSig2 module in wally's secp256k1-zkp (required for SuperScalar plugin).
# This adds --enable-module-musig to wally's secp256k1 configure so the musig
# headers are available at compile time.
sed -i 's/\[--enable-module-ecdsa-s2c\]/[--enable-module-ecdsa-s2c], [--enable-module-musig]/' \
  external/libwally-core/configure.ac

./configure
make -j$(nproc)
```

> **Why the `sed`?** CLN's wally bundles secp256k1-zkp but doesn't enable its MuSig2 module
> (wally doesn't need it). The SuperScalar plugin does — it uses MuSig2 for factory tree
> signing. Without this, the plugin can't compile against CLN's secp256k1 headers.

### Plugin Build

After building CLN, build the SuperScalar plugin using the script in [superscalar-cln](https://github.com/8144225309/superscalar-cln):

```bash
CLN_DIR=/path/to/cln-blip56 SS_DIR=/path/to/SuperScalar /path/to/superscalar-cln/build-plugin.sh
```

See [`superscalar-cln/build-plugin.sh`](https://github.com/8144225309/superscalar-cln/blob/main/build-plugin.sh) for full details.

## Files Changed vs Upstream

| File | Changes |
|------|---------|
| `wire/peer_wire.csv` | TLV 65600 (`channel_in_factory`) on `open_channel` and `accept_channel` |
| `common/features.h` | Feature bit 270/271 (`pluggable_channel_factories`) |
| `openingd/openingd.c` | TLV 65600 set/validate/echo, zero-conf enforcement |
| `lightningd/opening_control.c` | Factory info propagation, skip funding watch, `factory_funding_txid` override |
| `channeld/channeld_wire.csv` | Internal wire 7232/7233/7235/7236 (factory_change_init/locked/confirmed/abort) |
| `channeld/channeld.c` | STFU-gated factory-change, batch `commitment_signed` with factory inflight, `channel_update_funding` on lock |
| `lightningd/channel_control.c` | `factory-change`, `factory-forget-channel`, `checkutxo` RPCs; factory_change_locked handler |
| `wallet/wallet.c` | Crash fix for missing channels in coin movements |

---

Everything below this line is the standard Core Lightning documentation from upstream.

---

# Core Lightning (CLN): A specification compliant Lightning Network implementation in C

Core Lightning (previously c-lightning) is a lightweight, highly customizable and [standard compliant][std] implementation of the Lightning Network protocol.

* [Getting Started](#getting-started)
    * [Installation](#installation)
    * [Starting lightningd](#starting-lightningd)
    * [Using the JSON-RPC Interface](#using-the-json-rpc-interface)
    * [Care And Feeding Of Your New Lightning Node](#care-and-feeding-of-your-new-lightning-node)
    * [Opening A Channel](#opening-a-channel)
	* [Sending and Receiving Payments](#sending-and-receiving-payments)
	* [Configuration File](#configuration-file)
* [Further Information](#further-information)
    * [FAQ](doc/FAQ.md)
    * [Pruning](#pruning)
    * [HD wallet encryption](#hd-wallet-encryption)
	* [Developers](#developers)
* [Documentation](https://docs.corelightning.org/docs)

## Project Status

[![Continuous Integration][actions-badge]][actions]
[![Pull Requests Welcome][prs-badge]][prs]
[![Documentation Status][docs-badge]][docs]
[![Telegram][telegram-badge]][telegram]
[![Discord][discord-badge]][discord]
[![Irc][IRC-badge]][IRC]

This implementation has been in production use on the Bitcoin mainnet since early 2018, with the launch of the [Blockstream Store][blockstream-store-blog].
We recommend getting started by experimenting on `testnet` (`testnet4` or `regtest`), but the implementation is considered stable and can be safely used on mainnet.

## Reach Out to Us

Any help testing the implementation, reporting bugs, or helping with outstanding issues is very welcome.
Don't hesitate to reach out to us on the implementation-specific [mailing list][ml1], or on [CLN Discord][discord], or on [CLN Telegram][telegram], or on IRC at [dev][irc1]/[gen][irc2] channel.

## Getting Started

Core Lightning only works on Linux and macOS, and requires a locally (or remotely) running `bitcoind` (version 25.0 or above) that is fully caught up with the network you're running on, and relays transactions (ie with `blocksonly=0`).
Pruning (`prune=n` option in `bitcoin.conf`) is partially supported, see [here](#pruning) for more details.

### Installation

There are 3 supported installation options:

 - Installation of a pre-compiled binary from the [release page][releases] on GitHub.
 - Using one of the [provided docker images][dockerhub] on the Docker Hub.
 - Compiling the source code yourself as described in the [installation documentation](doc/getting-started/getting-started/installation.md).

### Starting `lightningd`

#### Regtest (local, fast-start) Option
If you want to experiment with `lightningd`, there's a script to set
up a `bitcoind` regtest test network of two local lightning nodes,
which provides a convenient `start_ln` helper. See the notes at the top
of the `startup_regtest.sh` file for details on how to use it.

```bash
. contrib/startup_regtest.sh
```

#### Mainnet Option
To test with real bitcoin,  you will need to have a local `bitcoind` node running:

```bash
bitcoind -daemon
```

Wait until `bitcoind` has synchronized with the network.

Make sure that you do not have `walletbroadcast=0` in your `~/.bitcoin/bitcoin.conf`, or you may run into trouble.
Notice that running `lightningd` against a pruned node may cause some issues if not managed carefully, see [below](#pruning) for more information.

You can start `lightningd` with the following command:

```bash
lightningd --network=bitcoin --log-level=debug
```

This creates a `.lightning/` subdirectory in your home directory: see `man -l doc/lightningd.8` (or https://docs.corelightning.org/docs) for more runtime options.

### Using The JSON-RPC Interface

Core Lightning exposes a [JSON-RPC 2.0][jsonrpcspec] interface over a Unix Domain socket; the `lightning-cli` tool can be used to access it, or there is a [python client library](contrib/pyln-client).

You can use `lightning-cli help` to print a table of RPC methods; `lightning-cli help <command>`
will offer specific information on that command.

Useful commands:

* [newaddr](doc/lightning-newaddr.7.md): get a bitcoin address to deposit funds into your lightning node.
* [listfunds](doc/lightning-listfunds.7.md): see where your funds are.
* [connect](doc/lightning-connect.7.md): connect to another lightning node.
* [fundchannel](doc/lightning-fundchannel.7.md): create a channel to another connected node.
* [invoice](doc/lightning-invoice.7.md): create an invoice to get paid by another node.
* [pay](doc/lightning-pay.7.md): pay someone else's invoice.
* [plugin](doc/lightning-plugin.7.md): commands to control extensions.

### Care And Feeding Of Your New Lightning Node

Once you've started for the first time, there's a script called
`contrib/bootstrap-node.sh` which will connect you to other nodes on
the lightning network.

There are also numerous plugins available for Core Lightning which add
capabilities: in particular there's a collection at: https://github.com/lightningd/plugins

For a less reckless experience, you can encrypt the HD wallet seed:
 see [HD wallet encryption](#hd-wallet-encryption).

You can also chat to other users at Discord [core-lightning][discord];
we are always happy to help you get started!


### Opening A Channel

First you need to transfer some funds to `lightningd` so that it can
open a channel:

```bash
# Returns an address <address>
lightning-cli newaddr
```

`lightningd` will register the funds once the transaction is confirmed.

Alternatively you can generate a taproot address should your source of funds support it:

```bash
# Return a taproot address
lightning-cli newaddr p2tr
```

Confirm `lightningd` got funds by:

```bash
# Returns an array of on-chain funds.
lightning-cli listfunds
```

Once `lightningd` has funds, we can connect to a node and open a channel.
Let's assume the **remote** node is accepting connections at `<ip>`
(and optional `<port>`, if not 9735) and has the node ID `<node_id>`:

```bash
lightning-cli connect <node_id> <ip> [<port>]
lightning-cli fundchannel <node_id> <amount_in_satoshis>
```

This opens a connection and, on top of that connection, then opens a channel.
The funding transaction needs 3 confirmation in order for the channel to be usable, and 6 to be announced for others to use.
You can check the status of the channel using `lightning-cli listpeers`, which after 3 confirmations (1 on testnet) should say that `state` is `CHANNELD_NORMAL`; after 6 confirmations you can use `lightning-cli listchannels` to verify that the `public` field is now `true`.

### Sending and Receiving Payments

Payments in Lightning are invoice based.
The recipient creates an invoice with the expected `<amount>` in
millisatoshi (or `"any"` for a donation), a unique `<label>` and a
`<description>` the payer will see:

```bash
lightning-cli invoice <amount> <label> <description>
```

This returns some internal details, and a standard invoice string called `bolt11` (named after the [BOLT #11 lightning spec][BOLT11]).

[BOLT11]: https://github.com/lightning/bolts/blob/master/11-payment-encoding.md

The sender can feed this `bolt11` string to the `decodepay` command to see what it is, and pay it simply using the `pay` command:

```bash
lightning-cli pay <bolt11>
```

Note that there are lower-level interfaces (and more options to these
interfaces) for more sophisticated use.

## Configuration File

`lightningd` can be configured either by passing options via the command line, or via a configuration file.
Command line options will always override the values in the configuration file.

To use a configuration file, create a file named `config` within your top-level lightning directory or network subdirectory
(eg. `~/.lightning/config` or `~/.lightning/bitcoin/config`).  See `man -l doc/lightningd-config.5`.

A sample configuration file is available at `contrib/config-example`.

## Further information

### Pruning

Core Lightning requires JSON-RPC access to a fully synchronized `bitcoind` in order to synchronize with the Bitcoin network.
Access to ZeroMQ is not required and `bitcoind` does not need to be run with `txindex` like other implementations.
The lightning daemon will poll `bitcoind` for new blocks that it hasn't processed yet, thus synchronizing itself with `bitcoind`.
If `bitcoind` prunes a block that Core Lightning has not processed yet, e.g., Core Lightning was not running for a prolonged period, then `bitcoind` will not be able to serve the missing blocks, hence Core Lightning will not be able to synchronize anymore and will be stuck.
In order to avoid this situation you should be monitoring the gap between Core Lightning's blockheight using `lightning-cli getinfo` and `bitcoind`'s blockheight using `bitcoin-cli getblockchaininfo`.
If the two blockheights drift apart it might be necessary to intervene.

### HD wallet encryption

You can encrypt the `hsm_secret` content (which is used to derive the HD wallet's master key) by passing the `--encrypted-hsm` startup argument, or by using the `lightning-hsmtool` (which you can find in the `tool/` directory at the root of this repo) with the `encrypt` method. You can unencrypt an encrypted `hsm_secret` using the `lightning-hsmtool` with the `decrypt` method.

If you encrypt your `hsm_secret`, you will have to pass the `--encrypted-hsm` startup option to `lightningd`. Once your `hsm_secret` is encrypted, you __will not__ be able to access your funds without your password, so please beware with your password management. Also, beware of not feeling too safe with an encrypted `hsm_secret`: unlike for `bitcoind` where the wallet encryption can restrict the usage of some RPC command, `lightningd` always needs to access keys from the wallet which is thus __not locked__ (yet), even with an encrypted BIP32 master seed.

### Developers

Developers wishing to contribute should start with the developer guide [here](doc/contribute-to-core-lightning/coding-style-guidelines.md).

## Related Projects

| Project | Description |
|---------|-------------|
| [SuperScalar](https://github.com/8144225309/SuperScalar) | Reference implementation of the SuperScalar protocol |
| [superscalar-cln](https://github.com/8144225309/superscalar-cln) | SuperScalar channel factory plugin for Core Lightning (bLIP-56) |
| [superscalar-wallet](https://github.com/8144225309/superscalar-wallet) | Web-based wallet UI for SuperScalar factory management |
| [superscalar-docs](https://github.com/8144225309/superscalar-docs) | Protocol documentation and visual guides |
| [superscalar.win](https://superscalar.win) | SuperScalar explainer and documentation site |

[blockstream-store-blog]: https://blockstream.com/2018/01/16/en-lightning-charge/
[std]: https://github.com/lightning/bolts
[prs-badge]: https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat
[prs]: http://makeapullrequest.com
[ml1]: https://lists.ozlabs.org/listinfo/c-lightning
[discord-badge]: https://badgen.net/badge/Discord/chat/blue
[discord]: https://discord.gg/mE9s4rc5un
[telegram-badge]: https://badgen.net/badge/Telegram/chat/blue
[telegram]: https://t.me/lightningd
[IRC-badge]: https://img.shields.io/badge/IRC-chat-blue.svg
[IRC]: https://web.libera.chat/#c-lightning
[irc1]: https://web.libera.chat/#lightning-dev
[irc2]: https://web.libera.chat/#c-lightning
[docs-badge]: https://readthedocs.org/projects/lightning/badge/?version=docs
[docs]: https://docs.corelightning.org/docs
[releases]: https://github.com/ElementsProject/lightning/releases
[dockerhub]: https://hub.docker.com/r/elementsproject/lightningd/
[jsonrpcspec]: https://www.jsonrpc.org/specification
[helpme-github]: https://github.com/lightningd/plugins/tree/master/helpme
[actions-badge]: https://github.com/ElementsProject/lightning/workflows/Continuous%20Integration/badge.svg
[actions]: https://github.com/ElementsProject/lightning/actions
