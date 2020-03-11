---
ics: 10
title: GRANDPA Client
stage: draft
category: IBC/TAO
kind: instantiation
author: Yuanchao Sun <ys@cdot.network>
created: 2020-03-15
modified: (modification date)
requires: (optional list of ics numbers)
required-by: (optional list of ics numbers)
implements: 2
---

## Synopsis

(high-level description of and rationale for specification)
GRANDPA (GHOST-based Recursive ANcestor Deriving Prefix Agreement) is a finality gadget that will be used in Polkadot relay chain.
GRANDPA now has a Rust implementation and is part of Substrate, so it's likely that blockchains built using Substrate will use GRANDPA as consensus engine.
This specification document describes a client (verification algorithm) for a blockchain using GRANDPA finality gadget.

### Motivation

(rationale for existence of standard)
Blockchains using GRANDPA finality gadget might like to interface with other replicated state machines or solo machines over IBC.

### Definitions

(definitions of any new terms not defined in common documentation)
Functions & terms are as defined in [ICS 2](../ics-002-client-semantics).

### Desired Properties

(desired characteristics / properties of protocol, effects if properties are violated)
This specification must satisfy the client interface defined in ICS 2.

## Technical Specification

(main part of standard document - not all subsections are required)

(detailed technical specification: syntax, semantics, sub-protocols, algorithms, data structures, etc)
This specification depends on correct instantiation of the [GRANDPA finality gadget](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf).

### Data Structures

(new data structures, if applicable)

### Client state

The GRANDPA client state tracks latest height and frozen height.

```typescript
interface ClientState {
  latestHeight: uint64
  frozenHeight: Maybe<uint64>
}
```

### Consensus state

The GRANDPA client tracks authority set and commitment root for all previously verified consensus states.

```typescript
type AuthorityId = Address
type AuthorityWeight = uint64
interface ConsensusState {
  setId: uint64
  authorities: List<Pair<AuthorityId, AuthorityWeight>>
  commitmentRoot: []byte
}
```

### Headers

The GRANDPA client headers include the height and the commitment root.

```typescript
interface Header {
  number: uint64
  commitmentRoot: []byte
}
```

### Commit
A commit message which is an aggregate of precommits.
```typescript
interface Commit {
	/// The target block's hash.
	targetHash: Hash
	/// The target block's number.
	targetNumber: uint64
	/// Precommits for target block or any block after it that justify this commit.
	precommits: []Signature
}
```

### Justification

A GRANDPA justification for block finality, it includes a commit message and an ancestry proof including all headers routing all precommit target blocks to the commit target block. Due to the current voting strategy the precommit targets should be the same as the commit target, since honest voters don't vote past authority set change blocks.

```typescript
interface Justification {
  round: uint64
  commit: Commit
  votesAncestries: []Header
}
```

### Evidence

The `Evidence` type is used for detecting misbehaviour and freezing the client - to prevent further packet flow - if applicable.
GRANDPA client `Evidence` consists of two headers at the same height both of which the light client would have considered valid.

```typescript
interface Evidence {
  fromHeight: uint64
  h1: Header
  h2: Header
}
```

### Client initialisation

GRANDPA client initialisation requires a (subjectively chosen) latest consensus state, including the full authorities.

```typescript
function initialise(setId: uint64, authorities: List<Pair<AuthorityId, AuthorityWeight>>, commitmentRoot: []byte, height: uint64): ClientState {
    assert(height > 0)
    consensusState = ConsensusState{setId, authorities, header.commitmentRoot}
    set("clients/{identifier}/consensusStates/{height}", consensusState)
    return ClientState{
      latestHeight: height,
      frozenHeight: null,
    }
}
```

The GRANDPA client `latestClientHeight` function returns the latest stored height, which is updated every time a new (more recent) header is validated.

```typescript
function latestClientHeight(clientState: ClientState): uint64 {
  return clientState.latestHeight
}
```

### Validity predicate

GRANDPA client validity checking verifies a header is signed by the current authority set and verifies the authorities_proof to determine if there is a expected change to the authority set. If the provided header is valid, the client state is updated & the newly verified commitment written to the store.

```typescript
function checkValidityAndUpdateState(
  clientState: ClientState,
  header: Header,
  justification: Justification,
  authoritiesProof: []byte) {
    // assert header height is newer than any we know
    assert(header.height > clientState.latestHeight)
    consensusState = get("clients/{identifier}/consensusStates/{clientState.latestHeight}")
    // call the `verifyJustification` function
    assert(justification.verify(consensusState.setId, consensusState.authorities))
    assert(justification.commit.targetHash == hash(header))
    // verify that the new authorities has been stored
    assert(header.commitmentRoot.verifyMembership(path, newAuthorities, authoritiesProof))
    // update latest height
    clientState.latestHeight = header.height
    // create recorded consensus state, save it
    consensusState = ConsensusState{newSetId, newAuthorities, header.commitmentRoot}
    set("clients/{identifier}/consensusStates/{header.height}", consensusState)
    // save the client
    set("clients/{identifier}", clientState)
}
```

### Misbehaviour predicate

GRANDPA client misbehaviour checking determines whether or not two conflicting headers at the same height would have convinced the light client.

```typescript
function checkMisbehaviourAndUpdateState(
  clientState: ClientState,
  evidence: Evidence) {
    // assert that the heights are the same
    assert(evidence.h1.height === evidence.h2.height)
    // assert that the commitments are different
    assert(evidence.h1.commitmentRoot !== evidence.h2.commitmentRoot)
    // fetch the previously verified commitment root & validator set
    consensusState = get("clients/{identifier}/consensusStates/{evidence.fromHeight}")
    // check if the light client "would have been fooled"
    assert(
      verify(consensusState.authorities, evidence.fromHeight, evidence.h1) &&
      verify(consensusState.authorities, evidence.fromHeight, evidence.h2)
      )
    // set the frozen height
    clientState.frozenHeight = min(clientState.frozenHeight, evidence.h1.height) // which is same as h2.height
    // save the client
    set("clients/{identifier}", clientState)
}
```

### State verification functions

GRANDPA client state verification functions check a Merkle proof against a previously validated commitment root.

```typescript
function verifyClientConsensusState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  clientIdentifier: Identifier,
  consensusStateHeight: uint64,
  consensusState: ConsensusState) {
    path = applyPrefix(prefix, "clients/{clientIdentifier}/consensusState/{consensusStateHeight}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided consensus state has been stored
    assert(root.verifyMembership(path, consensusState, proof))
}

function verifyConnectionState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  connectionIdentifier: Identifier,
  connectionEnd: ConnectionEnd) {
    path = applyPrefix(prefix, "connections/{connectionIdentifier}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided connection end has been stored
    assert(root.verifyMembership(path, connectionEnd, proof))
}

function verifyChannelState(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  channelEnd: ChannelEnd) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided channel end has been stored
    assert(root.verifyMembership(path, channelEnd, proof))
}

function verifyPacketData(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  data: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/packets/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided commitment has been stored
    assert(root.verifyMembership(path, hash(data), proof))
}

function verifyPacketAcknowledgement(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64,
  acknowledgement: bytes) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the provided acknowledgement has been stored
    assert(root.verifyMembership(path, hash(acknowledgement), proof))
}

function verifyPacketAcknowledgementAbsence(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  sequence: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/acknowledgements/{sequence}")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that no acknowledgement has been stored
    assert(root.verifyNonMembership(path, proof))
}

function verifyNextSequenceRecv(
  clientState: ClientState,
  height: uint64,
  prefix: CommitmentPrefix,
  proof: CommitmentProof,
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  nextSequenceRecv: uint64) {
    path = applyPrefix(prefix, "ports/{portIdentifier}/channels/{channelIdentifier}/nextSequenceRecv")
    // check that the client is at a sufficient height
    assert(clientState.latestHeight >= height)
    // check that the client is unfrozen or frozen at a higher height
    assert(clientState.frozenHeight === null || clientState.frozenHeight > height)
    // fetch the previously verified commitment root & verify membership
    root = get("clients/{identifier}/consensusStates/{height}")
    // verify that the nextSequenceRecv is as claimed
    assert(root.verifyMembership(path, nextSequenceRecv, proof))
}
```

### Sub-protocols

(sub-protocols, if applicable)

### Properties & Invariants

(properties & invariants maintained by the protocols specified, if applicable)
Correctness guarantees as provided by the GRANDPA light client algorithm.

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

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).
