## What is this

This proposes a new type of channel which has no on-chain funding but otherwise works according to LN rules. Notable features are the following:

* It defines two distinct roles: Client and Custodian. Client code is to be implemented in end-user LN wallets and Custodian is a service which is to be provided by willing businesses. Once implemented, any business who wishes to be a Custodian will be able to use this unified protocol instead of building their own ad-hoc custodial wallets from scratch.

* Funds in custodial channel is a pure Custodian's IOU to Client which is not automatically enforcable on-chain but is still auditable. Whereas normal channel provides enforcability by a means of cross-signed commitment transaction a custodial channel provides auditability by a notion of latest cross-signed state which can be used to prove Custodian's obligations.

* It provides privacy guarantees identical to normal channels: Custodian does not know who the Client is and has no metadata on Client's payments because route selection, onion formation as well as preimage generation for incoming payments still happen on Client side.

* The main motivation for this is radically simplified UX and zero on-chain footprint: custodial channels can be created on the fly with zero on-chain fund allocation on either Custodian or Client side. As such, custodial channels should be seen as an auditable and privacy preserving improvement over current custodial LN wallets; they may exist along with normal channels in otherwise non-custodial LN wallets and function as an easy on-ramp to LN for new users.

## Specifics

* Custodial channel is always private and does not route thrid-party payments. It can never be closed, its data may only be removed from Client wallet if there is no desire to use it anymore. Once removed, it can always be fully restored later provided that Client still has a wallet seed.

* Custodial channel ID is deterministic and is always known in advance for every Client/Custodian pair. It is derived as `sha256(bip69_lexicographical_order(Client node ID, Custodian node ID))`.

* A notion of _blockday_ is introduced as `current blockchain height / 144`, and used in messages specific to custodial channels, the intention is to ground them in real blockchain timeline.

* Custodial channel must be explicitly invoked by Client on reconnection which is different from normal channel reestablish procedure. Once invoked, Custodian should reply with either `init_custodial_channel` message if no channel exists yet or with `last_cross_signed_state` if it does.

* On each reconnection Custodian should send a `channel_update` message to Client as defined in BOLT7. `short_channel_id` field in that message does not point to any on-chain transaction but just serves as internal channel identifier and at once enables receiving of payments. `short_channel_id` must also have a block number which is equal or less than `500000` to never mix it with normal channels. `cltv_expiry_delta` must be set to at least `432` blocks and `fee_proportional_millionths`/`fee_base_msat` are advised to be higher than those set by Custodian for their normal channels.
  * The reason for higher `fee_proportional_millionths`/`fee_base_msat` is three-fold: (1) it gives Custodian an economic incentive to run a service, (2) gives Client an incentive to eventually move to normal channels and (3) pushes custodial channel to the end of the routing queue in case if receiving Client has mixed normal and custodial channels in their wallet, thus making normal channels more likely to receive a payment first.
  * The reason for higher `cltv_expiry_delta` is to give Client a time window to resolve possible receiving conflicts, more on this below.

## Message types

        +-------+                                           +-------+
        |       |--(1)------ invoke_custodial_channel ----->|       |
        |       |<-(2)------ init_custodial_channel --------|       |
        |       |--(2.1)---- state_signature -------------->|       |
        |       |<-(2.2)---- state_signature ---------------|       |
        |       |                  or                       |       |
        |   A   |<-(2)------ last_cross_signed_state -------|   B   |
        |       |--(3)------ update_add_htlc -------------->|       |
        |       |--(4)------ state_signature -------------->|       |
        |       |<-(5)------ state_signature ---------------|       |
        |       |<-(6)------ update_fulfill_htlc -----------|       |
        |       |<-(7)------ state_signature ---------------|       |
        |       |--(8)------ state_signature -------------->|       |
        +-------+                                           +-------+

        - where node A is Client and node B is Custodian

### The `invoke_custodial_channel` Message

1. type: 65535 (`invoke_custodial_channel`)
2. data:
  * [`32`:`chain_hash`]
  * [`2`:`len`]
  * [`len`:`refund_scriptpubkey`]

#### Requirements

`refund_scriptpubkey` requiements are similar to `scriptpubkey` in https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#closing-initiation-shutdown

#### Rationale

This message is sent by Client on each reconnection, it prompts Custodian to reveal last cross signed channel state or offer a new custodial channel if none exists for a given Client yet. `refund_scriptpubkey` should be persisted by Custodian and may be later used to refund current Client balance on-chain (for example, if Custodian decides to stop providing a service or if client is offline for long period of time). Client may change `refund_scriptpubkey` on each reconnection.

### The `init_custodial_channel` Message

1. type: 65534 (`init_custodial_channel`)
2. data:
  * [`8`:`max_htlc_value_in_flight_msat`]
  * [`8`:`htlc_minimum_msat`]
  * [`2`:`max_accepted_htlcs`]
  * [`8`:`channel_capacity_satoshis`]
  * [`2`:`liability_deadline_blockdays`]
  * [`8`:`minimal_onchain_refund_amount_satoshis`]
  * [`8`:`initial_client_balance_satoshis`]

#### Rationale

This message is sent by Custodian to Client in reply to `invoke_custodial_channel` if no custodial channel exists for a given Client yet. 

`liability_deadline_blockdays` specifies a period in blockdays after last `state_signature` exchange during which the Custodian is going to maintain a channel. That is, if there is no activity during `liability_deadline_blockdays` period then Custodian owes nothing to Client anymore.

`minimal_onchain_refund_amount_satoshis` specifies a minimal balance that Client should have in a custodial channel for Custodian to refund it on-chain using Client's `refund_scriptpubkey`.


### The `last_cross_signed_state` Message

1. type: 65533 (`last_cross_signed_state`)
2. data:
  * [`2`:`len`]
  * [`len`:`last_refund_scriptpubkey`]
  * [`44`:`init_custodial_channel`]
  * [`8`:`last_client_balance_satoshis`]
  * [`4`:`block_day`]
  * [`4`:`client_update_counter`]
  * [`4`:`custodian_update_counter`]
  * [`64`:`client_node_signature`]
  * [`64`:`custodian_node_signature`]

#### Rationale

This message is sent by Custodian to Client in reply to `invoke_custodial_channel` if a custodial channel already exists for a given Client.

Client must make sure that both `client_node_signature` and `custodian_node_signature` match and that `block_day`/`client_update_counter`/`custodian_update_counter` values are not lower than the ones Client currently has.

If both signatures match but `block_day`/`client_update_counter`/`custodian_update_counter` values are above the Client's values then Client updates it's internal channel state accoring to values from `last_cross_signed_state` message (this may happen if client has fallen behind or lost channel data).

Node signatures sign `sha256(refund_scriptpubkey + minimal_onchain_refund_amount_satoshis + liability_deadline_blockdays + client_balance + block_day + client_update_counter + custodian_update_counter)` with respected node private keys.

While verifying a signature a drift of 1 blockday is permitted (for example, if Custodian's current blockday is 100 and signature verification fails, then Custodian must retry it once with blockday set to 101).

### The `state_signature` Message

1. type: 65532 (`state_signature`)
2. data:
  * [`8`:`updated_client_balance_satoshis`]
  * [`4`:`block_day`]
  * [`4`:`client_update_counter`]
  * [`4`:`custodian_update_counter`]
  * [`2`:`num_client_htlcs`]
  * [`num_client_htlcs*44`:`client_outgoing_htlcs`]
  * [`2`:`num_custodian_htlcs`]
  * [`num_client_htlcs*44`:`custodian_outgoing_htlcs`]
  * [`64`:`node_signature`]

#### Rationale

Custodian and Client exchange `state_signature` messages after sending `update_add_htlc`/
`update_fail_htlc`/`update_fail_malformed_htlc`/`update_fulfill_htlc` messages, a state is considered cross-signed once both Custodian and Client valid signatures are collected for the same channel state.

`client_outgoing_htlcs` and `custodian_outgoing_htlcs` is a list of current in-flight HTLCs, each record is a tuple of `(payment_hash, amount_msat, cltv_expiry)` sorted according to rules defined at https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#transaction-input-and-output-ordering.

When sending `state_signature` a peer must increment its respected `update_counter`, when receiving a remote `state_signature` it must increment a remote `update_counter`. Thus `state_signature` is a CvRDT with defined merge operation which is guaranteed to eventually converge. Client and Custodian must keep `state_signature` exchange until convergence is achieved.

### The `state_override` Message

1. type: 65531 (`state_override`)
2. data:
  * [`8`:`updated_client_balance_satoshis`]
  * [`4`:`block_day`]
  * [`4`:`client_update_counter`]
  * [`4`:`custodian_update_counter`]
  * [`64`:`node_signature`]

#### Rationale

Either party may send a custodial channel into `SUSPENDED` state by sending out an `Error` message. Whereas normal channel gets force-closed a custodial one gets `SUSPENDED`. Normal operation may be resumed once that happens by Custodian sending a `state_override` message to Client which would erase a previous problematic state and set a new agreed upon Client's balance. Client must manually accept this message which would send a `state_override` in return. Client's wallet UI/UX must be especially explicit about what is going on in this situation.
