---
ics: 10
title: GRANDPA Client
stage: draft
category: IBC/TAO
kind: instantiation
author: Julian Sun <julian@oct.network>, Rivers Yang <rivers@oct.network>
created: 2020-03-15
modified: 2022-11-11
implements: 2
---

## Synopsis

This specification document describes a client (verification algorithm) for a blockchain using GRANDPA finality gadget with BEEFY protocol.

GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement) is the finality gadget that is implemented for the Polkadot Relay Chain. It now has a Rust implementation and is part of the Substrate. Solo chains built using Substrate may also use GRANDPA as its finality gadget.

[The BEEFY protocol](https://github.com/paritytech/grandpa-bridge-gadget/blob/master/docs/beefy.md) is designed to enable blockchains using GRANDPA finality gadget to be efficiently validated by light clients. Using this protocol to implement the GRANDPA client is the best option for now.

### Motivation

State machines of various sorts replicated using the GRANDPA finality gadget might like to interface with other replicated state machines or solo machines over IBC.

### Definitions

Functions & terms are as defined in [ICS 2](../../core/ics-002-client-semantics).

### Desired Properties

This specification must satisfy the client interface defined in ICS 2.

## Technical Specification

This specification depends on correct instantiation of the BEEFY protocol.

### Client state

The GRANDPA client state tracks the latest MMR root, the latest height when the MMR root is updated, a possible frozen height, the current and the next validator set.

```typescript
interface ClientState {
    latestMMRRoot: []byte
    latestHeight: uint64
    frozenHeight: Maybe<uint64>
    currentValidatorSet: ValidatorSet
    nextValidatorSet: ValidatorSet
}
```

### Validator set

The validator set of a GRANDPA client consists of the set id, a merkle root of validator addresses and the number of validator in the set.

```typescript
interface ValidatorSet {
  id: uint64
  root: []byte
  length: uint32
}
```

### Commitment

A commitment signed by GRANDPA validators as part of BEEFY protocol.
The commitment contains a payload extracted from the finalized block at
height block_number.
GRANDPA validators collect signatures on commitments and a stream of such signed commitments

```typescript
interface Commitment {
  payload: []byte
  blockNumber: uint32
  validatorSetId: uint64
}
```

### SignedCommitment

A commitment with validators' signatures

```typescript

type Signature = [65]byte

interface SignedCommitment {
  commitment: Commitment
  signatures: []Maybe<Signature>
}
```

### NewCommitment

```typescript
interface NewCommitment {
  signedCommitment: SignedCommitment
  validatorProofs: MerkleProof
  leaf: MMRLeaf
  proof: MMRProof
}
```

NewCommitment implements `ClientMessage` interface.

### Misbehaviour
 
The `Misbehaviour` type is used for detecting misbehaviour and freezing the client - to prevent further packet flow - if applicable.
GRANDPA client `Misbehaviour` consists of two signed commitments at the same height both of which the light client would have considered valid.

```typescript
interface Misbehaviour {
  fromHeight: uint64
  signedCommitment1: SignedCommitment
  validatorProofs1: MerkleProof
  signedCommitment2: SignedCommitment
  validatorProofs2: MerkleProof
}
```

Misbehaviour implements `ClientMessage` interface.

### Client initialisation

GRANDPA client initialisation requires a (subjectively chosen) latest height, including the full validator set at that height and the next validator set.

```typescript
function initialise(height: uint64, initialValidatorSet: ValidatorSet, nextValidatorSet: ValidatorSet): ClientState {
    assert(height > 0)
    return ClientState{
      latestMMRRoot: nil,
      latestHeight: height,
      frozenHeight: null,
      currentValidatorSet: initialValidatorSet,
      nextValidatorSet 
    }
}
```

The GRANDPA client `latestClientHeight` function returns the latest stored height, which is updated every time a new (more recent) MMR root is validated.

```typescript
function latestClientHeight(clientState: ClientState): Height {
  return clientState.latestHeight
}
```

### Validity predicate

GRANDPA client validity checking verifies a commitment is signed by the current validator set and verifies the authority set proof to determine if there is a expected change to the authority set. If the provided commitment is valid, the client state is updated & the newly verified commitment written to the store.

```typescript
function verifyClientMessage(
  clientMsg: ClientMessage) {
    switch typeof(clientMsg) {
      case NewCommitment:
        verifyCommitment(clientMsg.signedCommitment, clientMsg.validatorProofs)
        verifyMMR(clientMsg.signedCommitment.commitment.payload, clientMsg.leaf, clientMsg.proof)
      case Misbehaviour:
        verifyCommitment(clientMsg.signedCommitment1, clientMsg.validatorProofs1)
        verifyCommitment(clientMsg.signedCommitment2, clientMsg.validatorProofs2)
    }
}
```

```typescript
function verifyCommitment(signedCommitment: SignedCommitment, validatorProofs: []MerkleProof) {
    clientState = get("clients/{header.identifier}/clientState")
    // the block height of the new commitment must be greater than the latest height
    assert(signedCommitment.commitment.blockNumber > clientState.latestHeight)
    for _, signature := range signedCommitment.signatures {
      validator = secp256k1Recover(signature)
      assert(validator != null)
      found = false
      for _, proof := range validatorProofs {
        if proof.leaf == validator {
          found = true
          assert(verifyMerkleProof(clientState.currentValidatorSet.root, proof.leaf, proof.proof))
        }
      }
      assert(found)
    }
}
```

### Misbehaviour predicate

GRANDPA client misbehaviour checking determines whether or not two conflicting signed commitment at the same height would have convinced the light client.

```typescript
function checkForMisbehaviour(
  clientMsg: clientMessage) => bool {
    clientState = get("clients/{clientMsg.identifier}/clientState")
    switch typeof(clientMsg) {
      case Header:
      case Misbehaviour:
        // assert that the commitments are different
        assert(misbehaviour.signedCommitment1.commitment.payload !== misbehaviour.signedCommitment2.commitment.payload )
    }
}
```

### UpdateState

UpdateState will perform a regular update for the GRANDPA client. If the height of the new commitment is higher than the latest height on the clientState, then the clientState will be updated.

```typescript
function updateState(
  clientMsg: clientMessage) {
    clientState = get("clients/{clientMsg.identifier}/clientState)
    (commitment, nextValidatorSet) = NewCommitment(clientMessage)
    // only update the clientstate if the commitment height is higher
    // than clientState latest height
    if clientState.height < commitment.blockNumber {
      // update latest height
      clientState.latestHeight = commitment.blockNumber
      clientState.latestMMRRoot = commitment.payload
      if clientState.currentValidatorSet.id + 1 == nextValidatorSet.id {
        clientState.currentValidatorSet = beefyNextAuthoritySet
      }

      // save the client
      set("clients/{clientMsg.identifier}/clientState", clientState)
    }
}
```

### UpdateStateOnMisbehaviour

UpdateStateOnMisbehaviour will set the frozen height to a non-zero sentinel height to freeze the entire client.

```typescript
function updateStateOnMisbehaviour(clientMsg: clientMessage) {
    clientState = get("clients/{clientMsg.identifier}/clientState)
    misbehaviour = Misbehaviour(clientMessage)
    clientState.frozenHeight = misbehaviour.fromHeight
    set("clients/{clientMsg.identifier}/clientState", clientState)
}
```

### Upgrades
### State verification functions

GRANDPA client state verification functions check a Merkle proof against a previously validated commitment root.

These functions utilise the `proofSpecs` with which the client was initialised.

```typescript
function verifyMembership(
  clientState: ClientState,
  height: Height,
  proof: CommitmentProof,
  path: CommitmentPath,
  value: []byte) {
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    // Implementations may choose how to pass in the identifier
    // ibc-go provides the identifier-prefixed store to this method
    // so that all state reads are for the client in question
    root = get("clients/{clientIdentifier}/consensusStates/{height}")
    // verify that <path, value> has been stored
    assert(verifyMembership(root, proof, path, value))
}

function verifyNonMembership(
  clientState: ClientState,
  height: Height,
  proof: CommitmentProof,
  path: CommitmentPath) {
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    // Implementations may choose how to pass in the identifier
    // ibc-go provides the identifier-prefixed store to this method
    // so that all state reads are for the client in question
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that nothing has been stored at path
    assert(verifyMembership(root, proof, path))
}
```

### Properties & Invariants

Correctness guarantees as provided by the BEEFY protocol.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Not applicable. Alterations to the client verification algorithm will require a new client standard.

## Example Implementation

None yet.

## Other Implementations

None at present.

## History

March 15, 2020 - Initial version
November 11, 2022 - Update based on BEEFY protocol

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
