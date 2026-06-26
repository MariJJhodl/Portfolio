# RGB on Lightning: What's Coming Next

**SEO title:** RGB Lightning Roadmap: Implementation and What's Next
**Keyphrase:** RGB Lightning roadmap
**Meta description:** The RGB Lightning roadmap explained: OP_RETURN commitments, the LDK fork, rgb-lightning-node, and what comes next.
**Slug:** rgb-lightning-roadmap
**Author:** RGB Team
**Featured image alt text:** Zoe Fatilbà presenting the RGB Lightning roadmap at Tuscany Lightning Summit 2026 in Viareggio

---

*This article is based on Zoe Fatilbà's talk on the RGB Lightning roadmap at the Tuscany Lightning Summit 2026, held in Viareggio in May 2026.*

---

**Key takeaways:**
- RGB Protocol on Bitcoin integrates with the Lightning Network by adding an OP_RETURN commitment to every channel transaction — from funding to closing — inheriting all Lightning guarantees unchanged.
- RGB Lightning channels always carry both satoshis and assets: satoshis must be above the dust limit to keep outputs spendable, but don't need to be economically significant.
- [rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node) (RLN) is the first implementation, built on rgb-lib and a fork of LDK, with a REST API and a live Regtest demo via [@rgb_lightning_bot](https://t.me/rgb_lightning_bot).
- The roadmap covers privacy improvements (SPV proofs, Taproot channels), new payment capabilities (multi-asset payments, BOLT12), and safety-critical fixes (dynamic fee rates, CPFP).
- Zoe invited contributors to find a good first issue and contribute to RGB development.

---

## How Does RGB Protocol Work on the Lightning Network?

RGB Protocol on Bitcoin works through **[client-side validation](https://rgb.info/learn/client-side-validation/)**: asset data never touches the chain, and only the parties involved in a transfer validate the relevant history. 

As Zoe put it:

> "We have an architecture very similar to a peer-to-peer network, where you only need to validate the history that's relevant for you, there's no global state."

The key mechanism on-chain is an **OP_RETURN** output added to Bitcoin transactions, which commits to an [RGB state transition](https://docs.rgb.info/rgb-state-and-operations/state-transitions) without exposing any asset data publicly.

Bringing this approach to the RGB Lightning Network required applying the same principle to every transaction in a channel's lifecycle, from funding to closing.

> "We just need to take every Lightning transaction, from the funding to the closing, and add an OP_RETURN output to it. By doing so, we inherit all the guarantees the Lightning Network gives, like penalty transactions, HTLC timeouts, and so on."

There is one practical constraint: whenever RGB assets need to move across a channel, the **[commitment transaction](https://docs.rgb.info/commitment-layer/commitment-schemes)** must include an **HTLC** output. This mechanism requires sending a bitcoin amount alongside the asset to keep the output spendable. The satoshis don't need to be economically significant, but they must be sufficient to keep all commitment outputs above dust. RGB channels therefore always carry both satoshis and assets, which also ensures the Lightning penalty mechanism remains intact.

To know more about RGB Protocol on Lightning Network, [see the related page](https://rgb.info/learn/rgb-lightning-network/).


---

## RLN: The First RGB-Enabled Lightning Node

[rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node) (RLN) is the first Lightning node with native RGB support. Built on rgb-lib and a fork of [LDK](https://lightningdevkit.org/) (Lightning Development Kit), it extends the standard Lightning stack to handle RGB assets: opening channels that carry both satoshis and RGB assets, routing asset payments across the network, and managing the off-chain RGB state in sync with channel state. 

It exposes a REST API, so any application can interact with it without needing to understand the underlying protocol details.

To get started:

```bash
git clone https://github.com/RGB-Tools/rgb-lightning-node
```

The README includes instructions to launch three nodes on Regtest. There is also a live demo available via the Telegram [@rgb_lightning_bot](https://t.me/rgb_lightning_bot) running on a shared Regtest environment, which lets you receive free testnet Bitcoin and RGB assets, then pay a Lightning invoice to receive a sticker.

Zoe was direct about the invitation to contribute:

> "It's not that hard to add RGB to a Lightning node. I encourage everybody who would like to try, to do so, because the more, the better."

---

## What Had to Change in LDK

The LDK fork required targeted modifications to the protocol messages and transaction construction logic, but the difference is relatively small, and Zoe **encouraged anyone interested to read through it directly**. The modifications are more surgical than foundational.

The changes fall into two categories:

- **Transaction coloring**: for every transaction the node creates (commitment, HTLC, and closing), the node checks whether the channel carries RGB assets and, if so, colors it by adding an OP_RETURN output that commits to the corresponding RGB state transition.

- **Modified TLV messages**: the LDK fork extends four Lightning messages:
  - `open_channel` — the initiator announces where the **consignment** can be downloaded
  - `update_add_htlc` — carries the asset ID and amount so all peers along the route know what is being transferred
  - `channel_announcement` — lets nodes advertise which asset a channel supports, for gossip and routing
  - `channel_update` — intermediary nodes announce the maximum RGB amount they are willing to route

On the counterparty side, before creating the funding transaction, the node downloads and validates the consignment. If it is invalid, the node discards the funding, ensuring both parties have verified the asset history before any on-chain action takes place. A recent addition also allows the channel initiator to push assets to the counterparty at open, mirroring the `push_msat` mechanism in standard Lightning.

---

## What's on the RGB Lightning Network Roadmap?

Zoe outlined planned work across three areas: the core RGB protocol, rgb-lib, and RLN.

### Protocol

The two main threads on the protocol side are privacy and infrastructure hardening. 

On privacy: **SPV** (Simplified Payment Verification) proofs let the node verify transactions by querying block headers directly, rather than sending full data to an indexer. This approach removes a significant source of transfer pattern leakage.

On the consignment side, consignment creation will be improved to conceal every RGB output that is not relevant to the recipient or needed to allow safe validation — reducing the information a transfer reveals to third parties about unrelated outputs.

Beyond privacy, a new schema — **BFA** — is in the works to enable bridging assets from other chains into RGB via burn and bridge transitions, opening a migration path for assets issued elsewhere.
Also, **consignment streaming** will optimize memory usage on mobile: today, the full consignment must be loaded into memory before validation begins; the planned approach validates incrementally and discards already-processed history, keeping memory usage bounded.

Finally, the remaining items are prerequisites for institutional adoption: replacing the current custom file format with proper database storage, expanding documentation and unit test coverage, and completing internal and external audits.

### rgb-lib

[rgb-lib](https://github.com/RGB-Tools/rgb-lib) is the Rust library that makes RGB accessible to application developers. Bindings for Python, Swift, Kotlin, and Node.js let developers build RGB applications without writing Rust. 

The current priorities centre on reliability and ecosystem reach.

On reliability: a **journaling system** combined with database transactions will ensure that failures roll back state atomically, eliminating a class of inconsistency errors. 
Reorg detection is not yet in development, but the approach is already clear: save the last synced block hash on shutdown and verify it is still on-chain on restart. If a reorg has affected a validated transfer, an existing API call rolls back the RGB state.
The team also plans **RBF** (Replace-By-Fee) support.

On ecosystem reach: the team is actively improving Node.js bindings to reach first-class support. Bear runtime bindings will enable **integration of rgb-lib into the Tether Wallet Development Kit (WDK)**. Additionally, **WASM** (WebAssembly) support will allow rgb-lib to run in browser environments.

The proxy, i.e. the off-chain service that mediates consignment exchange between sender and receiver, will also improve. The team is adding **receiver authentication via public key**, so only the legitimate recipient can accept a transfer. Additionally, senders will be able to pre-sign and post the transaction to the proxy. The receiver can then broadcast it independently, without waiting for the sender to come back online.

### RGB Lightning Node (RLN)

The RLN roadmap spans several dimensions: active development, safety fixes, privacy improvements, and new payment capabilities.

On the infrastructure side, active work is focused on three fronts:

- enabling atomic swaps with external systems via **HODL invoices**
- making the node lighter and easier to run **without a full Bitcoin Core connection**
- supporting high-availability deployments through **remote signing**, so a remote host can operate the node without holding signing authority over the funds

Zoe flagged two items as safety-critical. 

First, dynamic fee rates: the node currently uses a fixed fee rate, which could affect fund safety on channel close. 
Second, **CPFP** (Child Pays For Parent) support: the node already uses anchoring outputs, but cannot yet bump their fees if commitment transactions land on-chain with insufficient fees.

On the privacy side, three improvements are in the works:

- **Taproot channels** will remove the current ability of external observers to detect that a channel carries RGB assets
- **dynamic blinding factor** will replace the current static blinding factor, which reduces privacy if a channel is force-closed
- **exchanging consignments over the Lightning peer-to-peer** network, rather than through the proxy, will eliminate the proxy's privacy implications entirely

In terms of payment capabilities, **multi-asset channels** and multi-asset payments are both on the roadmap. Multi-asset payments are particularly notable:

> "I don't have USDT, I have just Bitcoin and I want to pay this invoice. I can find a route where in the middle there's a swap node, and I will be able to pay whatever asset even if I don't own it."

As a consequence, a user holding only BTC could pay a USDT-denominated invoice, with the asset conversion handled automatically along the route. **Splicing** will allow adding RGB assets to a vanilla Bitcoin channel — a standard Lightning channel carrying only satoshis — in a single transaction.

Zoe also plans dual funded channels and **BOLT12** support — the latter being **a good first contribution for anyone looking to get involved**.

The longer-term architectural goal is to upstream the RGB-specific LDK changes directly, eliminating the need to maintain a fork. Before reaching that goal, the team is actively improving error handling and test coverage — particularly for penalty transactions and compatibility with standard Lightning nodes.

---
## Getting Involved

The RGB stack is actively developed and open to contributors. As Zoe made clear during the talk, the changes required to add RGB support to a Lightning node are smaller than they might appear. Moreover, the team has marked specific issues as **good entry points for new contributors**, including BOLT12 support for RLN.

If you want to explore the code, run a local Regtest setup, or open a PR, the repositories are here:

- **rgb-lightning-node**: [github.com/RGB-Tools/rgb-lightning-node](https://github.com/RGB-Tools/rgb-lightning-node)
- **rgb-lib**: [github.com/RGB-Tools/rgb-lib](https://github.com/RGB-Tools/rgb-lib)
- **RGB protocol**: [github.com/rgb-protocol](https://github.com/rgb-protocol)
- **Documentation**: [docs.rgb.info](https://docs.rgb.info)

[@rgb_lightning_bot](https://t.me/rgb_lightning_bot) runs on a shared Regtest environment and lets you receive testnet Bitcoin and RGB assets and pay a Lightning invoice without setting up anything locally. 

Give it a try!
