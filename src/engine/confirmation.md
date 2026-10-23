# Engine API -- Confirmed Block Specification

Engine API method that passes the block confirmed by the consensus layer's fast confirmation rule to the execution layer, so that the execution layer can serve it under the `confirmed` block tag.

## Table of contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Confirmed Block Specification](#confirmed-block-specification)
- [Methods](#methods)
  - [engine_confirmedBlockUpdatedV1](#engine_confirmedblockupdatedv1)
    - [Request](#request)
    - [Response](#response)
    - [Specification](#specification)
- [Rationale](#rationale)
- [Open questions](#open-questions)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Confirmed Block Specification

The [fast confirmation rule](https://github.com/ethereum/consensus-specs/blob/master/specs/phase0/fast-confirmation.md) lets a consensus layer client confirm a block, usually within one or two slots, under synchrony and honesty assumptions that differ from those behind the existing `safe` and `finalized` tags. The method in this document lets the consensus layer pass that block to the execution layer, which serves it under the `confirmed` block tag of the `eth` namespace.

Support is optional. An execution layer client that implements the method always accepts it, without any configuration. A consensus layer client calls it only if its operator has enabled it. Support is independent of network upgrades: it does not change `ForkchoiceState` or any `engine_forkchoiceUpdated` version, and it does not change the meaning of the `safe` or `finalized` tags. Nodes whose operators do not enable it behave exactly as before.

## Methods

### engine_confirmedBlockUpdatedV1

#### Request

* method: `engine_confirmedBlockUpdatedV1`
* params:
  1. `confirmedBlockHash`: `DATA`, 32 Bytes - block hash of the most recent execution block confirmed by the consensus layer's fast confirmation rule
* timeout: 1s

#### Response

* result: `null`
* error: code and message set in case an exception happens while processing the request.

#### Specification

1. Execution layer client software **MAY** support this method. If it does, it **MUST** include `engine_confirmedBlockUpdatedV1` in its [`engine_exchangeCapabilities`](./common.md#engine_exchangecapabilities) response, and **MUST NOT** require any configuration to accept it or to serve the `confirmed` block tag.

2. Consensus layer client software **MUST NOT** call this method unless the node operator has enabled it and the execution layer client has included `engine_confirmedBlockUpdatedV1` in its `engine_exchangeCapabilities` response.

3. Consensus layer client software that has this method enabled **SHOULD** call it each time the execution block hash derived from its fast-confirmed block changes, and **MUST** call it only after the `engine_forkchoiceUpdated` call that makes `confirmedBlockHash` equal to or an ancestor of `headBlockHash`. This includes changes to an older block, for example when the fast confirmation rule falls back to the finalized block.

4. Client software **MUST** return `-38001: Unknown payload` if the block referenced by `confirmedBlockHash` is not known, and **MUST** return `-38002: Invalid forkchoice state` if it is known but is not equal to or an ancestor of the current head of the canonical chain. In both cases the execution layer client **MUST NOT** change its confirmed block.

5. Otherwise, the execution layer client **MUST** set its confirmed block to the block referenced by `confirmedBlockHash`, even if that block is older than the previous confirmed block, and return `null`.

6. The execution layer client **MUST** resolve the `confirmed` block tag to its confirmed block while that block is in the canonical chain. If no confirmed block has been set since the client started, or the confirmed block is no longer in the canonical chain, the `confirmed` tag **MUST** be treated as a tag that cannot be resolved to a block, in the same way as a `safe` or `finalized` tag that cannot be resolved. The execution layer client **MUST NOT** substitute another block, such as the `safe`, `finalized` or `latest` block.

7. This method **MUST NOT** affect fork choice, the `safe` and `finalized` blocks, or payload building. Execution layer client software **MUST NOT** log any error messages if this method has never been called or hasn't been called for a significant amount of time.

## Rationale

- *A separate method instead of a new `ForkchoiceState` field.* `engine_exchangeCapabilities` negotiates methods, not fields. A separate method can always be enabled in execution layer clients that implement it, so only the consensus layer operator has to turn the feature on. It works as soon as both clients support it, without a new `engine_forkchoiceUpdated` version or a network upgrade, and leaves every other node unchanged.
- *A new tag instead of changing `safe`.* Some applications, RPC providers and layer 2 networks rely on `safe` returning a justified block. A separate tag lets them adopt fast confirmations without breaking those users. Serving both from the same node also requires the consensus layer to send the justified block as `safeBlockHash`, which is outside the scope of this document.
- *No substitution.* Never answering `confirmed` with another block means that a node which has not enabled the feature, or has just restarted, cannot be mistaken for one that serves fast confirmations.
- *The confirmed block can move backwards.* The fast confirmation rule falls back to the finalized block when its checks fail. Applications should not treat the highest `confirmed` block they have seen as irreversible.

## Open questions

- *Coupling with `engine_forkchoiceUpdated`.* Rules 3 and 4 tie this method to `engine_forkchoiceUpdated`: the consensus layer calls it only after the `engine_forkchoiceUpdated` call that makes `confirmedBlockHash` an ancestor of the head, and the execution layer rejects a block that is not an ancestor of its current head. Client teams are asked which of these they prefer:
  - *Strict (current text).* The execution layer never stores a confirmed block outside the canonical chain, and the consensus layer learns about mismatches. The consensus layer must order fast confirmation updates after fork choice updates, and a re-org between the two calls can cause errors that are not real failures.
  - *Loose.* Drop the ordering requirement in rule 3 and the ancestor check in rule 4. The execution layer accepts any known block and stores it, and rule 6 makes the `confirmed` tag resolve only while that block is in the canonical chain. The two calls become independent, and the execution layer still never serves a block outside the canonical chain. A wrong hash is accepted without an error, and the `confirmed` tag can be briefly unresolvable until the head catches up.
