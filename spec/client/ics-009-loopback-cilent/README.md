---
ics: 9
title: Loopback Client
stage: draft
category: IBC/TAO
kind: instantiation
author: Christopher Goes <cwgoes@tendermint.com>
created: 2020-01-17
modified: 2023-03-02
requires: 2
implements: 2
---

## Synopsis

This specification describes a loopback client, designed to be used for interaction over the IBC interface with modules present on the same ledger.

### Motivation

Loopback clients may be useful in cases where the calling module does not have prior knowledge of where precisely the destination module lives and would like to use the uniform IBC message-passing interface (similar to `127.0.0.1` in TCP/IP).

### Definitions

Functions & terms are as defined in [ICS 2](../../core/ics-002-client-semantics).

`ConnectionEnd` and `generateIdentifier` are as defined in [ICS 3](../../core/ics-003-connection-semantics)

`getCommitmentPrefix` and `removePrefix` are as defined in [ICS 24](../../core/ics-024-host-requirements).

### Desired Properties

Intended client semantics should be preserved, and loopback abstractions should be negligible cost.

## Technical Specification

### Data Structures

No consensus state, headers, or evidence data structures are required for a loopback client. The loopback client does not need to store the consensus state of a remote chain, since state verification does not require to check a Merkle proof against a previously validated commitment root.

```typescript
type ConsensusState object

type Header object

type Misbehaviour object
```

### Client state

The loopback client state tracks the latest height of the local ledger.

```typescript
interface ClientState {
  latestHeight: Height
}
```

### Height

The height of a loopback client state consists of two `uint64`s: the revision number, and the height in the revision.

```typescript
interface Height {
  revisionNumber: uint64
  revisionHeight: uint64
}
```

### Sentinel objects

Similarly as in TCP/IP, where there exists a single loopback address, the protocol defines the existence of a single sentinel `ClientState` instance with the client identifier `09-localhost`. Creation of other loopback clients MUST be forbidden.

Additionally, implementations **will** reserve a special connection identifier `connection-localhost` to be used by a single sentinel `ConnectionEnd` stored by default (i.e. at genesis or upgrade). The `clientIdentifier` and `counterpartyClientIdentifier` of the connection end both reference the sentinel `09-localhost` client identifier. The `counterpartyConnectionIdentifier` references the special connection identifier `connection-localhost`. The existence of a sentinel loopback connetion end enables IBC applications to establish channels directly on top of the sentinel connection. Channel handshakes can then be initiated by supplying the loopback connection identifier (`connection-localhost`) in the `connectionHops` parameter of the `ChanOpenInit` datagram.

Implementations **may** also allow the creation of more connections associated with the loopback client. These connections would then have a connection identifier as generated by `generateIdentifier`.

### Relayer messages

Relayers supporting localhost packet flow must be adapted to submit messages from sending applications back to the originating chain.

This would require first checking the underlying connection identifier on any channel-level messages. If the underlying connection identifier is `connection-localhost`, then the relayer must construct the message and send it back to the originating chain. The message MUST be constructed with a sentinel byte for the proof (`[]byte{0x01}`), since the loopback client does not need Merkle proofs of the state of a remote ledger; the proof height in the message may be zero, since it is ignored by the loopback client.

Implementations **may** choose to implement loopback such that the next message in the handshake or packet flow is automatically called without relayer-driven transactions. However, implementors must take care to ensure that automatic message execution does not cause gas consumption issues.

### Client initialisation

Loopback client initialisation requires the latest height of the local ledger.

```typescript
function initialise(height: Height): ClientState {
  assert(height > 0)
  return ClientState{
    latestHeight: height
  }
}
```

### Validity predicate

No validity checking is necessary in a loopback client; the function should never be called.

```typescript
function verifyClientMessage(clientMsg: ClientMessage) {
  assert(false)
}
```

### Misbehaviour predicate

No misbehaviour checking is necessary in a loopback client; the function should never be called.

```typescript
function checkForMisbehaviour(clientMsg: clientMessage) => bool {
  return false
}
```

### Update state

Function `updateState` will perform a regular update for the loopback client. The `clientState` will be updated with the lastest height of the local ledger. This function should be called automatically at every height.

```typescript
function updateState(clientMsg: clientMessage) {
  clientState = provableStore.get("clients/{clientMsg.identifier}/clientState")

  // retrieve the latest height from the local ledger
  height = getSelfHeight()
  clientState.latestHeight = header.height

  // save the client state
  provableStore.set("clients/{clientMsg.identifier}/clientState", clientState)
}
```

### Update state on misbehaviour

Function `updateStateOnMisbehaviour` is unsupported by the loopback client and performs a no-op.

```typescript
function updateStateOnMisbehaviour(clientMsg: clientMessage) { }
```

### State verification functions

State verification functions simply need to read state from the local ledger and compare with the bytes stored under the standardized key paths. The loopback client needs read-only access to the **entire IBC store** of the local ledger, and not only to its own client identifier-prefixed store.

```typescript
function verifyMembership(
  clientState: ClientState,
  height: Height,
  delayTimePeriod: uint64,
  delayBlockPeriod: uint64,
  proof: CommitmentProof,
  path: CommitmentPath,
  value: []byte
): Error {
  // path is prefixed with the store prefix of the commitment proof
  // e.g. in ibc-go implementation this is "ibc"
  // since verification is done on the IBC store of the local ledger
  // the prefix needs to be removed from the path to retrieve the
  // correct key in the store
  unprefixedPath = removePrefix(getCommitmentPrefix(), path)
  
  // The complete (not only client identifier-prefixed) store is needed
  // to  verifiy that a path has been set to a particular value
  if provableStore.get(unprefixedPath) !== value {
    return error
  }
  return nil
}

function verifyNonMembership(
  clientState: ClientState,
  height: Height,
  delayTimePeriod: uint64,
  delayBlockPeriod: uint64,
  proof: CommitmentProof,
  path: CommitmentPath
): Error {
  // path is prefixed with the store prefix of the commitment proof
  // e.g. in ibc-go implementation this is "ibc"
  // since verification is done on the IBC store of the local ledger
  // the prefix needs to be removed from the path to retrieve the
  // correct key in the store
  unprefixedPath = removePrefix(getCommitmentPrefix(), path)

  // The complete (not only client identifier-prefixed) store is needed
  // to  verifiy that a path has not been set to a particular value
  if provableStore.get(unprefixedPath) !== nil {
    return error
  }
  return nil
}
```

### Properties & Invariants

Semantics are as if this were a remote client of the local ledger.

## Backwards Compatibility

Not applicable.

## Forwards Compatibility

Not applicable. Alterations to the client algorithm will require a new client standard.

## Example Implementations

- Implementation of ICS 09 in Go can be found in [ibc-go repository](https://github.com/cosmos/ibc-go). 

## History

January 17, 2020 - Initial version

March 2, 2023 - Update after ICS 02 refactor and addition of sentinel objects.

## Copyright

All content herein is licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0).