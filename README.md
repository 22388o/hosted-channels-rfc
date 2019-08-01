## What is this

This proposes a new type of channel which has no on-chain funding but otherwise works according to LN rules. Notable features are the following:

* It defines two distinct roles: Client and Host. Client code is to be implemented in end-user LN wallets and Host is a service which is to be provided by willing businesses. Once implemented, any business who wishes to be a Host will be able to use this unified protocol instead of building their own ad-hoc hosted wallets from scratch.

* Funds in hosted channel is a pure Host's IOU to Client which is not automatically enforcable on-chain but is still auditable. Whereas normal channel provides enforcability by a means of cross-signed commitment transaction a hosted channel provides auditability by a notion of latest cross-signed state which can be used to prove Host's obligations.

* It provides privacy guarantees identical to normal channels: Host does not know who the Client is and has no metadata on Client's payments because route selection, onion formation as well as preimage generation for incoming payments still happen on Client side.

* The main motivation for this is radically simplified UX and zero on-chain footprint: hosted channels can be created on the fly with zero on-chain fund allocation on either Host or Client side. As such, hosted channels should be seen as an auditable and privacy preserving improvement over current hosted LN wallets; they may exist along with normal channels in otherwise autonomous LN wallets and function as an easy on-ramp to LN for new users.

## Specifics

* Hosted channel is always private and does not route thrid-party payments. It can never be closed, its data may only be removed from Client wallet if there is no desire to use it anymore. Once removed, it can always be fully restored later provided that Client still has a wallet seed.

* Hosted channel ID is deterministic and is always known in advance for every Client/Host pair. It is derived as `sha256(bip69_lexicographical_order(Client node ID, Host node ID))`.

* A notion of _blockday_ is introduced as `current blockchain height / 144`, and used in messages specific to hosted channels, the intention is to ground them in real blockchain timeline.

* Hosted channel must be explicitly invoked by Client on reconnection which is different from normal channel reestablish procedure. Once invoked, Host should reply with either `init_hosted_channel` message if no channel exists yet or with `last_cross_signed_state` if it does.

* On each reconnection Host should send a `channel_update` message to Client as defined in BOLT7. `short_channel_id` field in that message does not point to any on-chain transaction but just serves as internal channel identifier and at once enables receiving of payments. `short_channel_id` must also have a block number which is equal or less than `100000` to never mix it with normal channels. `cltv_expiry_delta` must be set to at least `432` blocks and `fee_proportional_millionths`/`fee_base_msat` are advised to be higher than those set by Host for their normal channels.
  * The reason for higher `fee_proportional_millionths`/`fee_base_msat` is three-fold: (1) it gives Host an economic incentive to run a service, (2) gives Client an incentive to eventually move to normal channels and (3) pushes hosted channel to the end of the routing queue in case if receiving Client has mixed normal and hosted channels in their wallet, thus making normal channels more likely to receive a payment first.
  * The reason for higher `cltv_expiry_delta` is to give Client a time window to resolve possible receiving conflicts, more on this below.

## Message types

        +-------+                                           +-------+
        |       |--(1)------ invoke_hosted_channel -------->|       |
        |       |<-(2)------ init_hosted_channel -----------|       |
        |       |--(2.1)---- state_update ----------------->|       |
        |       |<-(2.2)---- state_update ------------------|       |
        |       |                  or                       |       |
        |   A   |<-(2)------ last_cross_signed_state -------|   B   |
        |       |--(3)------ update_add_htlc -------------->|       |
        |       |--(4)------ state_update ----------------->|       |
        |       |<-(5)------ state_update ------------------|       |
        |       |<-(6)------ update_fulfill_htlc -----------|       |
        |       |<-(7)------ state_update ------------------|       |
        |       |--(8)------ state_update ----------------->|       |
        +-------+                                           +-------+

        - where node A is Client and node B is Host

### The `invoke_hosted_channel` Message

1. type: 65535 (`invoke_hosted_channel`)
2. data:
  * [`chain_hash`:`chain_hash`]
  * [`u16`:`len`]
  * [`len*byte`:`refund_scriptpubkey`]

#### Requirements

`refund_scriptpubkey` requiements are similar to `scriptpubkey` in https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#closing-initiation-shutdown

#### Rationale

This message is sent by Client on each reconnection, it prompts Host to reveal last cross signed channel state or offer a new hosted channel if none exists for a given Client yet. `refund_scriptpubkey` should be persisted by Host and may be later used to refund current Client's balance on-chain (for example, if Host decides to stop providing a service or if client is offline for long period of time). Client may change `refund_scriptpubkey` on each reconnection.

### The `init_hosted_channel` Message

1. type: 65534 (`init_hosted_channel`)
2. data:
  * [`u64`:`max_htlc_value_in_flight_msat`]
  * [`u64`:`htlc_minimum_msat`]
  * [`u16`:`max_accepted_htlcs`]
  * [`u64`:`channel_capacity_satoshis`]
  * [`u16`:`liability_deadline_blockdays`]
  * [`u64`:`minimal_onchain_refund_amount_satoshis`]
  * [`u64`:`initial_client_balance_satoshis`]

#### Rationale

This message is sent by Host to Client in reply to `invoke_hosted_channel` if no hosted channel exists for a given Client yet. 

`liability_deadline_blockdays` specifies a period in blockdays after last `state_update` exchange during which the Host is going to maintain a channel. That is, if there is no activity during `liability_deadline_blockdays` period then Host owes nothing to Client anymore.

`minimal_onchain_refund_amount_satoshis` specifies a minimal balance that Client should have in a hosted channel for Host to refund it on-chain using Client's `refund_scriptpubkey`.


### The `last_cross_signed_state` Message

1. type: 65533 (`last_cross_signed_state`)
2. data:
  * [`u16`:`len`]
  * [`len*byte`:`last_refund_scriptpubkey`]
  * [[`init_hosted_channel`](#the-init_hosted_channel-message):`init_hosted_channel`]
  * [[`state_update`](#the-state_update-message):`last_client_state_update`]
  * [[`state_update`](#the-state_update-message):`last_host_state_update`]

#### Rationale

This message is sent by Host to Client in reply to `invoke_hosted_channel` if a hosted channel already exists for a given Client.

Client must make sure that `last_client_state_update` and `last_host_state_update` signatures are valid, and that `updated_client_balance_satoshis`/`block_day`/`client_update_counter`/`host_update_counter` are identical in both inner messages, and that actual values are not lower than the ones Client currently has locally.

If both signatures are valid but `block_day`/`client_update_counter`/`host_update_counter` values are above the Client's values then Client updates it's internal channel state accoring to values from `last_cross_signed_state` message (this may happen if client has fallen behind or lost channel data).

Node signatures sign `sha256(refund_scriptpubkey + liability_deadline_blockdays + minimal_onchain_refund_amount_satoshis + channel_capacity_satoshis + initial_client_balance_satoshis + updated_client_balance_satoshis + block_day + client_update_counter + host_update_counter + client_outgoing_htlcs + host_outgoing_htlcs)` with respected node private keys.

### The `state_update` Message

1. type: 65532 (`state_update`)
2. data:
  * [[`state_override`](#the-state_override-message):`state_override`]
  * [`u16`:`num_in_flight_htlcs`]
  * [`num_client_htlcs*53`:`in_flight_htlcs`]

#### Rationale

Host and Client exchange `state_update` messages after sending `update_add_htlc`/
`update_fail_htlc`/`update_fail_malformed_htlc`/`update_fulfill_htlc` messages, a state is considered cross-signed once both Host and Client valid signatures are collected for the same channel state.

`client_outgoing_htlcs` and `host_outgoing_htlcs` is a list of current in-flight HTLCs, each record is a tuple of `(from_host: Boolean, htlc_id, amount_msat, payment_hash, cltv_expiry)` sorted according to rules defined at https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#transaction-input-and-output-ordering.

When sending `state_update` a peer must increment its respected `update_counter`, when receiving a remote `state_update` it must increment a remote `update_counter`. Thus `state_update` is a CvRDT with defined merge operation which is guaranteed to eventually converge. Client and Host must keep exchanging `state_update` messages until convergence is achieved.

While verifying a signature a drift of 1 blockday is permitted i.e. `abs(ourBlockDay - theirBlockDate) <= 1`.

### The `state_override` Message

1. type: 65531 (`state_override`)
2. data:
  * [`u64`:`updated_client_balance_satoshis`]
  * [`u32`:`block_day`]
  * [`u32`:`client_update_counter`]
  * [`u32`:`host_update_counter`]
  * [`signature`:`node_signature`]

#### Rationale

Either party may send a hosted channel into `SUSPENDED` state by sending out an `Error` message. Whereas normal channel gets force-closed a hosted one gets `SUSPENDED`. Normal operation may be resumed once that happens by Host sending a `state_override` message to Client which would erase a previous problematic state and set a new agreed upon Client's balance. Client must manually accept this message which would send a `state_override` in return. Client's wallet UI/UX must be especially explicit about what is going on in this situation.

### Resolving edge cases

When establishing a hosted channel Client and Host agree that:

- The latest cross-signed state reflects Host's obligation to Client. Latest is defined as having the highest `client_update_counter`/`host_update_counter` combination since those values are only allowed to rise while `state_update` messages are exchanged.

- Host's obligation only lasts for `liability_deadline_blockdays` specified in `init_hosted_channel` message. For example: if last cross-signed `state_update` messages had blockday set to 1000 and `liability_deadline_blockdays` is set to 2000, then Host may not maintain a channel after blockday 3000 if it was not used all this time.

#### Host misses a channel

__Issue__: Client has a hosted channel with cross-signed state on their device but in response to `invoke_hosted_channel` Host replies with `last_cross_signed_state` message where `client_update_counter`/`host_update_counter` are below Client's values or with `init_hosted_channel` (offering to create a new channel) while channel has not expired yet according to `liability_deadline_blockdays` parameter.

__Solution__: Client puts channel to `SUSPENDED` state by sending an `Error`, then proivdes a latest cross-signed state to Host and/or third pary auditor. If Host can not provide a `last_cross_signed_state` with identical or higher `client_update_counter`/`host_update_counter` values then Host's debt is proven.

#### Client misses a channel

__Issue__: Client has fallen behind (by failing to save the last cross-signed state to disk), removed an app or completely lost their device and backups. Client still has a wallet seed (for example, in a form of mnemonic code) and rememebers who the Host is so can initiate a connection.

__Solution__: Client connects and sends `invoke_hosted_channel`, gets `last_cross_signed_state` in response, verifies both signatures and simply updates its channel state according to values from Host's message. It is possible for Host to lie to Client by providing an earlier and more favorable cross-signed state, but generally Host can not know that this exact Client has lost its state so Host would have to constantly probe this possibility by sending an old states to arbitrary Clients thus getting its channels `SUSPENDED` all the time.

#### Client stops responding in a middle of receiving

__Issue__: Malicious or offline Client may accept `update_add_htlc` and following `state_update` from Host without ever replying. In this situation Host has a limited time to either fulfill a payment downstream (which would require obtaining a preimage from Client) or fail it, otherwise Host risks downstream channel getting force-closed by remote peer.

__Solution__: Similar to how this is handled in normal channels, right before CLTV timelock is about to expire Host must put a hosted channel into `SUSPENDED` mode and fail a payment downstream. Hosted channel can be put back to operational mode at a later time by exchaning `state_override` messages where client balance is adjusted such that failed in-flight HTLC is removed.

#### Host stops responding in a middle of Client receiving

__Issue__: When Client receives a payment through hosted channel it sends `update_fulfill_htlc` with a preimage in response to Host's `update_add_htlc`. Host can use that preimage to propagate it back to payer and get the funds without sending updated `state_update` to Client. Host would thus get the funds into its normal channel while Client won't have a last cross-signed state reflecting that.

__Solution__: This situation will appear as highly unusual to Client: incoming payment will be seen as pending in Client's wallet and payer will be claiming that payment has been sent successfully while being able to provide a preimage: this would prove that Host has in fact obtained a preimage from Client and used it to increase its balance. Client would have the latest cross-signed state which fixates an incoming HTLC from Host along with CLTV expiry (which must be set to at least `432` blocks accoring to `channel_update` requirements above). Large CLTV expiry would give Client enough time to provide that cross-signed state along with payment preimage to Host and/or third pary auditor and thus prove Host's debt. Client must make this claim before CLTV is expired since after `432` blocks Host may put a channel into `SUSPENDED` mode and claim that Client was not responding. To further emphasize a critical nature of this requirement a Client's wallet may specifically distinguish those pending incoming payments for which `update_fulfill_htlc` has been sent but no `state_update` received yet).

#### Host decides to refund a channel on-chain using Client's `refund_scriptpubkey`

__Issue__: After doing that Client may show up with last cross-signed state claiming that funds are still in a hosted channel.

__Solution__: Host should first put channel into `SUSPENDED` state and then wait at least 1 blockday until broadcasting an on-chain refunding transaction, in that case it will be included in a block whose blockday is higher than last cross-signed state blockday, thus proving that Host has no debt anymore.
