# RGB on Lightning: Inside the Implementation and What's Coming Next

*Notes from Zoe Fatilbà's talk at the Tuscany Lightning Summit 2026*

---

Zoe Fatilbà is a developer at Bitfinex working on the RGB stack — from protocol-level fixes and testing to [rgb-lib](https://github.com/RGB-Tools/rgb-lib), the library that makes RGB accessible to application developers, and [rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node) (RLN), the first RGB-enabled Lightning node. At the Tuscany Lightning Summit 2026 in Viareggio, she walked through how RGB integrates with the Lightning Network at the implementation level, what the team changed in LDK to make it work, and what comes next.

This article summarises her talk, with direct quotes from her presentation.

---

## How RGB Works on Lightning

RGB on Bitcoin works through client-side validation: asset data never touches the chain, and only the parties involved in a transfer validate the relevant history. As Zoe put it:

> "We have an architecture very similar to a peer-to-peer network where you only need to validate the history that's relevant for you — there's no global state."

The key mechanism on-chain is an OP_RETURN output added to Bitcoin transactions. This output commits to an RGB state transition without exposing any asset data publicly.

Bringing this to Lightning required applying the same principle to every transaction in a channel's lifecycle — from funding to closing.

> "We just need to take every Lightning transaction, from the funding to the closing, and add an OP_RETURN output to it. By doing so, we inherit all the guarantees the Lightning Network gives — penalty transactions, HTLC timeouts, and so on."

There is one practical constraint: whenever RGB assets need to move across a channel, an HTLC output must be created in the commitment transaction. This means moving a small amount of satoshis alongside the asset — above the dust limit — to keep the output spendable. RGB channels therefore always carry both satoshis and assets, which also provides the economic basis for the Lightning penalty mechanism.

---

## RLN: The First RGB-Enabled Lightning Node

[rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node) (RLN) is built on rgb-lib and a fork of LDK (Lightning Development Kit). It exposes a REST API and can be used to open RGB channels, send asset payments, and interact with the network.

To get started:

```bash
git clone https://github.com/RGB-Tools/rgb-lightning-node
```

The README includes instructions to launch three nodes on Regtest. There is also a live demo available via a Telegram bot running on a shared Regtest environment — it lets you receive free testnet Bitcoin and RGB assets, then pay a Lightning invoice to receive a sticker.

Zoe was direct about the invitation to contribute:

> "It's not that hard to add RGB to a Lightning node. I encourage everybody who would like to try to do so, because the more the better."

---

## What Had to Change in LDK

The LDK fork required targeted modifications to the protocol messages and transaction construction logic. The key changes:

**Transaction coloring.** Every transaction the node creates — commitment transactions, HTLC transactions, closing transactions — is checked for whether the channel carries RGB assets. If so, the transaction is "colored" by adding an OP_RETURN output that commits to the corresponding RGB state transition.

**Modified TLV messages.** Four Lightning messages were extended:
- `open_channel` — the initiator announces the endpoint where the consignment can be downloaded
- `update_add_htlc` — extended to carry the asset ID and amount for routing
- `channel_announcement` — extended for gossip, so nodes can advertise which asset a channel supports
- `channel_update` — intermediary nodes announce the maximum RGB amount they're willing to route

**Consignment handling on channel open.** Before the funding transaction is created, the channel acceptor downloads and validates the consignment. If it's invalid, the acceptor discards the funding. This ensures both parties validate asset history before any on-chain action.

**Asset push on channel creation.** A recent addition allows the initiator to push assets to the counterparty during channel opening, mirroring the `push_msat` mechanism in standard Lightning.

The diff between the LDK fork and upstream is relatively small. Zoe encouraged anyone interested to read through the changes at the repository link — the modifications are more surgical than foundational.

---

## The Roadmap: Three Fronts

Zoe outlined planned work across three areas: the core RGB protocol, rgb-lib, and RLN.

### Protocol

**SPV proofs.** Currently, for every transaction in an asset's history, the node queries an indexer to confirm the transaction is on-chain. This exposes transfer patterns to the indexer. With SPV proofs, the node will query only block headers, making it significantly harder to correlate RGB transfers.

**Concealed seals.** An RGB state transition currently reveals all RGB outputs it creates. It will become possible to hide outputs not relevant to the recipient, improving privacy for multi-output transitions.

**BFA schema.** A new schema — Bridge Fungible Asset — will allow assets to be brought from other chains into RGB, via burn and bridge transitions. This is aimed at enabling existing assets on other networks to migrate to the RGB stack.

**Consignment streaming.** Today, the full consignment (the complete asset history relevant to a transfer) must be loaded into memory before validation. On resource-constrained devices like mobile, this is problematic. The planned improvement processes and validates the consignment incrementally, discarding already-validated history as it goes.

**Database storage, code cleanup, audits.** The current custom file format for RGB state will move to a proper database. Alongside this: documentation, unit tests, and internal and external audits — prerequisites before institutions put significant capital into RGB.

### rgb-lib

**Error handling and recovery.** A journaling system and database transactions will ensure that failures roll back state atomically.

**Node.js bindings.** Existing bindings exist for Python, Swift, and Node.js. The Node.js bindings are currently unreliable and will be promoted to first-class support.

**Bear runtime bindings.** This will enable RGB integration into the Tether Wallet Development Kit (WDK).

**WASM support.** Adds the ability to run rgb-lib in browser environments.

**Reorg detection and handling.** Not yet in development, but the approach is already mapped out: save the last synced block hash on shutdown, verify it's still on-chain on restart. Reorgs affecting already-validated transfers can then be detected and the RGB state rolled back via an existing API call.

**RBF support.** Replace-By-Fee is not yet supported.

**Proxy improvements.** The proxy is the off-chain service that mediates consignment exchange between sender and receiver. Planned improvements include:
- Authenticating the receiver via their public key, so only the legitimate recipient can accept a transfer
- Allowing the sender to pre-sign and post the transaction to the proxy — the receiver can then broadcast it directly without waiting for the sender to come back online

### RLN

**HODL invoices.** Three open PRs are adding HODL invoice support, which enables atomic swaps — not just between Lightning nodes, but with external systems as well.

**Transaction sync.** The node currently syncs the full chain header history from genesis. With transaction-level sync, it will query the indexer only for relevant transactions, allowing it to run on lighter hardware without a full Bitcoin Core connection.

**Multi-asset channels.** Currently each channel supports a single asset. Multi-asset channels will improve liquidity and routing for RGB payments.

**Multi-asset payments.** One of the more striking features in the roadmap:

> "I don't have USDT, I have just Bitcoin and I want to pay this invoice. I can find a route where in the middle there's a swap node, and I will be able to pay whatever asset even if I don't own it."

This means a user holding only BTC could pay a USDT-denominated invoice, with the conversion handled automatically along the route.

**Taproot channels.** RGB already supports both OP_RETURN and Taproot anchoring. Switching to Taproot channels will remove the current ability of external observers to infer that a channel carries RGB assets.

**Splicing.** In the RGB context, splicing has a specific value: it allows adding assets to an existing vanilla Bitcoin channel with a single transaction, without closing and reopening it.

**Dynamic blinding factors.** The current blinding factor used to conceal RGB output information is static — if a channel goes on-chain, the blinding factor becomes visible. Switching to a shared secret between channel counterparties removes this information leak.

**BOLT12.** Support for vanilla payments via BOLT12 is already partially in place. Zoe noted this as a good entry point for new contributors:

> "This is actually a good first issue. If anybody would like to jump on this, we will be very happy."

---

## On Trade-offs

A question during Q&A asked about the trade-offs of client-side validation. Zoe was direct:

> "The applications you can do with client-side validation are more restricted, because having a global state that everybody knows is something that can unlock some use cases. That's the biggest trade-off — it's not impossible to simulate a global state, but there's not actually a global view of the asset, and this limits some use cases."

On backups: RGB has a backup requirement similar to Lightning — losing state can result in losing assets. Zoe noted that on Lightning this is already a hard requirement, so RGB adds no fundamentally new obligation for node operators who already handle Lightning backups correctly.

---

## Getting Involved

- **rgb-lightning-node**: [github.com/RGB-Tools/rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node)
- **rgb-lib**: [github.com/RGB-Tools/rgb-lib](https://github.com/RGB-Tools/rgb-lib)
- **RGB protocol**: [github.com/rgb-protocol](https://github.com/rgb-protocol)
- **Documentation**: [docs.rgb.info](https://docs.rgb.info)
- **Telegram bot demo**: available via the RLN repository README

