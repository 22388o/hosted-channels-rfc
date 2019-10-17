## What is it

This proposes a new type of channel which has no on-chain funding but otherwise works according to LN rules. Notable features are the following:

* It defines two distinct roles: Client and Host. Client code is to be implemented in end-user LN wallets and Host is a service which is to be provided by willing businesses. Once implemented, any business who wishes to be a Host will be able to use this unified protocol instead of building their own ad-hoc hosted wallets from scratch.

* Funds in hosted channel is a pure Host's IOU to Client which is not automatically enforcable on-chain but is still auditable. Whereas normal channel provides enforcability by a means of cross-signed commitment transaction a hosted channel provides auditability by a notion of latest cross-signed state which can be used to prove Host's obligations.

* It provides privacy guarantees identical to normal channels: Host does not know who the Client is and has no metadata on Client's payments because route selection, onion formation as well as preimage generation for incoming payments still happen on Client side.

* The main motivation for this is radically simplified UX and zero on-chain footprint: hosted channels can be created on the fly with zero on-chain fund allocation on either Host or Client side. As such, hosted channels should be seen as an auditable and privacy preserving improvement over current hosted LN wallets; they may exist along with normal channels in otherwise autonomous LN wallets and function as an easy on-ramp to LN for new users.

## Specifics

* Hosted channel is always private and can never be closed in normal channel sense, its data may only be removed from Client wallet if there is no desire to use it anymore. Once removed, it can always be fully restored later provided that Client still has a wallet seed.

* Since hosted channels have no on-chain basis their `shortChannelId`s can not be derived from blockchain, as such they must be random and follow procedure defined in [#681](https://github.com/lightningnetwork/lightning-rfc/pull/681).

* Hosted channel ID is deterministic and is always known in advance for every Client/Host pair, it is derived as `sha256(bip69_lexicographical_order(Client node ID, Host node ID))`. This implies that each Client/Host pair may only have one hosted channel between them.

* A notion of _blockday_ is introduced as `current blockchain height / 144` to be used in messages specific to hosted channels, the intention is to ground them in real blockchain timeline.

## Invoking

* Hosted channel must be explicitly invoked by Client on reconnection which is different from normal channel reestablish procedure, the rationale is that Client may have normal channels with Host and may not wish to use a hosted one. Once invoked, Host should reply with either `init_hosted_channel` message if no channel exists yet or with `last_cross_signed_state` if it does.

        +-------+                                           +-------+
        |       |--(1)------ invoke_hosted_channel -------->|       |
        |       |<-(2)------ init_hosted_channel -----------|       |
        |       |--(3)------ state_update ----------------->|       |
        |       |<-(4)------ state_update ------------------|       | // new channel established
        |       |                  or                       |       |
        |   A   |<-(2)------ last_cross_signed_state -------|   B   |
        |       |--(3)------ last_cross_signed_state ------>|       | // existing channel invoked
        +-------+                                           +-------+

        - where node A is Client and node B is Host

### The `invoke_hosted_channel` Message

1. type: 65535 (`invoke_hosted_channel`)
2. data:
  * [`chain_hash`:`chain_hash`]
  * [`u16`:`len`]
  * [`len*byte`:`refund_scriptpubkey`]
  * [`u16`:`len`]
  * [`len*byte`:`secret`]
  
#### Rationale

* By sending this message Client prompts Host to reveal last cross signed channel state or offer a new hosted channel if none exists for a given Client yet.
* `refund_scriptpubkey` is similar to `scriptpubkey` in [02-peer-protocol#closing-initiation-shutdown](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#closing-initiation-shutdown) and must be provided by Client for a case when Host would want to stop a hosted channel and refund the rest of the balance while Client is offline.
* `secret` is an optional data which can be used by Host to tweak channel parameters (non-zero initial Client balance, larger capacity, only allow Clients with secrets etc).

### The `init_hosted_channel` Message

1. type: 65534 (`init_hosted_channel`)
2. data:
  * [`u64`:`max_htlc_value_in_flight_msat`]
  * [`u64`:`htlc_minimum_msat`]
  * [`u16`:`max_accepted_htlcs`]
  * [`u64`:`channel_capacity_msat`]
  * [`u16`:`liability_deadline_blockdays`]
  * [`u64`:`minimal_onchain_refund_amount_satoshis`]
  * [`u64`:`initial_client_balance_msat`]

#### Rationale

* This message is sent by Host in reply to `invoke_hosted_channel` if no hosted channel exists for a given Client yet. 
* `liability_deadline_blockdays` specifies a period in blockdays after last `state_update` exchange during which the Host is going to maintain a channel. That is, if there are no payments during `liability_deadline_blockdays` period then Host owes nothing to Client anymore.
* `minimal_onchain_refund_amount_satoshis` specifies a minimal balance that Client must have in a hosted channel for a Host to consider refunding it on-chain using Client's `refund_scriptpubkey`.

### The `last_cross_signed_state` Message

1. type: 65533 (`last_cross_signed_state`)
2. data:
  * [`u16`:`len`]
  * [`len*byte`:`last_refund_scriptpubkey`]
  * [[`init_hosted_channel`](#the-init_hosted_channel-message):`init_hosted_channel`]
  * [`u32`:`block_day`]
  * [`u64`:`local_balance_msat`]
  * [`u64`:`remote_balance_msat`]
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`u16`:`num_incoming_htlcs`]
  * [`num_incoming_htlcs*[update_add_htlc](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc)`:`incoming_htlcs`]
  * [`u16`:`num_outgoing_htlcs`]
  * [`num_outgoing_htlcs*[update_add_htlc](https://github.com/lightningnetwork/lightning-rfc/blob/master/02-peer-protocol.md#adding-an-htlc-update_add_htlc)`:`outgoing_htlcs`]
  * [`signature`:`remote_sig_of_local`]
  * [`signature`:`local_sig_of_remote`]

#### Rationale

* This message is sent by Host in reply to `invoke_hosted_channel` if it already exists for a given Client, once sent Host expects a similar `last_cross_signed_state` message from Client.
* Both parties must make sure that `remote_sig_of_local` and `local_sig_of_remote` signatures are valid, and that inverted `local_updates`/`remote_updates` numbers from remote `last_cross_signed_state` are the same as `local_updates`/`remote_updates` from local `last_cross_signed_state`.
* If local signature of remote `last_cross_signed_state` is valid but inverted `local_updates`/`remote_updates` numbers from remote `last_cross_signed_state` are higher than `local_updates`/`remote_updates` from local `last_cross_signed_state` then this means that local peer has fallen behind (or completely lost a state). Once this happens local peer must act as follows:
  * First, it must check if remote `last_cross_signed_state` points to one of future channel states which can be re-created locally. To enable this both peers store each local and remote `update_add_htlc`, `update_fulfill_htlc`, `update_fail_htlc`, `update_fail_malformed_htlc` message they send or receive in historic order until new `last_cross_signed_state` is reached. This vector of updates must be traversed with respected local `last_cross_signed_state` message created on each step and then compared against remote `last_cross_signed_state` update numbers. Once a match is found local peer applies it as current state, resolves all updates preceding this state and re-transmits all local updates following this state.
  * Second, if no matching future local `last_cross_signed_state` could be found then this means that local peer has completely lost it, in this case local peer should invert a remote `last_cross_signed_state` and apply it as its current state, then resolve all incoming HTLCs contained in remote `last_cross_signed_state`.
* Local state signature is created by signing `sha256(refund_scriptpubkey + liability_deadline_blockdays + minimal_onchain_refund_amount_satoshis + channel_capacity_msat + initial_client_balance_msat + block_day + local_balance_msat + remote_balance_msat + local_updates + remote_updates + incoming_htlcs + outgoing_htlcs)` fields taken from local `update_fail_malformed_htlc` with respected `nodeId` private key.

## Normal operation

* First a peer sends `update_add_htlc`/`update_fulfill_htlc`/`update_fail_htlc`/`update_fail_malformed_htlc` messages which are then followed by local `state_update` message, which is then followed by remote `state_update`. Once both valid `state_update`s are collected a new cross-signed state is reached which in turn allows peers to resolve previous updates.

        +-------+                                           +-------+
        |       |----------- update_add_htlc #1 ----------->|       |
        |       |----------- state_update ----------------->|       | // B has new local `last_cross_signed_state` #1, can start resolving #1
        |       |<---------- state_update ------------------|       | // A has new local `last_cross_signed_state` #1
        |   A   |<---------- update_fulfill_htlc #1 --------|   B   | 
        |       |<---------- state_update ------------------|       | // A has new local `last_cross_signed_state` #2
        |       |----------- state_update ----------------->|       | // B has new local `last_cross_signed_state` #2
        +-------+                                           +-------+

        Where B is Host and A is Client

### The `state_update` Message

1. type: 65532 (`state_update`)
2. data:
  * [`u32`:`block_day`]
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`signature`:`local_sig_of_remote`]

## Failure and state overriding

Normal channel operation may be interrupted in a number of ways (incorrect state update numbers, signature, timed out outgoing HTLC etc). Once this happens a peer must put a channel in `SUSPENDED` state and send out an `Error` message. Hosted channel use tagged errors with first two bytes of `Error` message reserved for the following cases:

`0001`: Wrong blockday in a remote message.  
`0002`: Wrong local signature from remote message.  
`0003`: Wrong remote signature from remote message.  
`0004`: CLTV delay in `channel_update` is too low.  
`0005`: Too many `state_update` messages without reaching of new local `last_cross_signed_state` (more than 16 in a row).  
`0006`: Timed out outgoing HTLC.  
`0007`: Remote peer has lost all channels and can't resolve in-flight HTLCs.  
`0008`: Hosted channel denied by Host when Client was trying to invoke it.  

Normal operation may be resumed after channel gets `SUSPENDED` by Host sending a `state_override` message to Client which would erase all previous problematic state and set a new agreed upon Client's balance. Client must manually accept this message which would send a `state_update` in return. Client's wallet UI/UX must be especially explicit about what is going on in this situation.

        +-------+                                           +-------+
        |       |<---------- state_override ----------------|       | // A has new local `last_cross_signed_state`
        |   A   |----------- state_update ----------------->|   B   | // B has new local `last_cross_signed_state`
        +-------+                                           +-------+

        Where B is Host and A is Client

### The `state_override` Message

1. type: 65531 (`state_override`)
2. data:
  * [`u32`:`block_day`]
  * [`u64`:`local_balance_msat`]
  * [`u32`:`local_updates`]
  * [`u32`:`remote_updates`]
  * [`signature`:`local_sig_of_remote`]

## Resolving edge cases

When establishing a hosted channel Client and Host agree that:

- The latest cross-signed state reflects Host's obligation to Client. Latest is the one having the highest `local_updates`/`remote_updates` combination since those values are only allowed to rise while `state_update` messages are exchanged.

- Host's obligation only lasts for `liability_deadline_blockdays` specified in `init_hosted_channel` message. For example: if `last_cross_signed_state` had `blockday` set to 1000 and `liability_deadline_blockdays` is set to 2000, then Host may not maintain a channel after blockday 3000 if it was not used all this time.
