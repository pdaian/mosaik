# Mosaik Byzantine Hardening Plan

## Objective

Harden `mosaik` from an honest-member runtime into a maximally defensive control plane that remains correct under Byzantine peers, stale leaders, replayed metadata, and malicious topology updates.

## Key Constraint

`mosaik` does not become Byzantine hard solely by adding TEEs.

TEEs can help with:

- node identity protection
- release integrity and admission control
- sealed keys and measured execution
- operator-to-node trust reduction

TEEs do not replace:

- Byzantine quorum protocols
- quorum certificates
- equivocation detection
- replay protection
- deterministic replicated state transitions

## What Mosaik Needs For Byzantine Hardness

### 1. Replace trusted-member assumptions with explicit BFT state

Every control-plane action that changes shared state must require Byzantine quorum agreement.

Required changes:

- replace or isolate Raft-backed authority paths with a BFT replication layer
- represent committed decisions as quorum-certified log entries
- require monotonic view numbers, epochs, and commit certificates
- reject any gossip-announced state that lacks a valid certificate

Candidate scope:

- subcluster manifests
- membership changes
- release policy
- round-set activation and drain commands
- admission and fencing state

### 2. Make gossip informational, never authoritative

Discovery gossip should advertise candidate facts only.

Required changes:

- treat peer announcements as hints pending certificate-backed confirmation
- separate observed catalog state from committed control state
- prevent automatic topology changes from raw gossip alone
- verify all cross-node metadata against signed manifests and quorum certificates

### 3. Add strong identity, versioning, and anti-replay rules

Required controls:

- stable node identities with signed membership records
- epoch fencing for operator actions and subcluster lifecycle
- per-message nonces or sequence numbers on control RPCs
- hash chaining for manifest revisions and policy bundles
- rejection of stale release IDs, stale attestation evidence, and stale epochs

### 4. Enforce deterministic state-machine behavior

BFT replication is only safe if each node applies the same input identically.

Required changes:

- define canonical serialization for all replicated control objects
- remove nondeterministic local inputs from replicated transitions
- treat wall-clock time as advisory only; commit explicit deadlines as data
- make conflict resolution deterministic and testable

### 5. Add Byzantine admission policy

Membership must require more than knowing a network secret.

Required changes:

- signed enrollment records issued by an operator or enrollment authority
- role-scoped authorization for producer, consumer, control, and worker identities
- attestation verification where TEE-backed roles are required
- release-policy checks so only approved binaries can participate in critical groups

## Where TEEs Help In Mosaik

TEEs are useful for reducing trust in the host and for strengthening admission.

Recommended use:

- store node identity and RPC signing keys inside the TEE
- attest the control-plane binary and policy hash at join time
- attest worker-side policy enforcement before accepting it into a BFT-managed subcluster
- seal crash-recovery cursors and last committed epoch locally

Limitations:

- a compromised but correctly attested node can still equivocate at the network level unless protocol messages are certificate-checked
- TEEs do not provide quorum intersection or Byzantine consensus by themselves
- discovery, pub/sub, and collections still need protocol-level BFT rules

## Subsystem Hardening Plan

### Discovery

- require signed peer records and short-lived freshness windows
- gate bootstrap peers behind enrollment policy
- classify discovered peers as `observed` until BFT-committed

### Streams

- add authenticated channels with sender identity bound to each datum
- support optional quorum-acknowledged delivery for control-critical streams
- rate-limit and isolate misbehaving publishers

### Collections and groups

- do not treat current Raft-backed groups as Byzantine-safe
- introduce a BFT-backed authority store for control-critical collections
- require quorum certificates on reads of safety-critical objects
- fence stale writers and reject conflicting leaders

### Runtime and APIs

- split operator API from gossip/discovery API
- require signed, versioned commands for all mutating admin actions
- add audit logging for every committed change and rejected conflicting change

## Codebase Work Plan

### Core protocol layer

- add a BFT control-log abstraction rather than overloading existing gossip state
- define certificate types for proposals, commits, and membership snapshots
- surface epoch and certificate validation in public APIs

### Collections integration

- route safety-critical objects through the BFT control log
- keep eventually consistent catalogs only for non-authoritative observability data

### Admission and identity

- add enrollment record verification hooks
- add attestation verification hooks for TEE-gated roles
- bind role tags to signed policy, not self-declared metadata

### Testing

- Byzantine tests for equivocation, forged announcements, stale leaders, and replayed membership updates
- deterministic-state tests across nodes
- partition and recovery tests ensuring no stale control state can re-activate drained resources
- TEE admission tests for measurement allowlist and downgrade rejection

## Release Gates

Do not call Mosaik Byzantine-hardened until:

- authoritative control state is committed by a BFT protocol
- raw gossip cannot mutate safety-critical state
- all critical control actions carry signatures, epochs, and replay protection
- deterministic application of committed control entries is proven by tests
- TEE admission, if enabled, is an additive gate and not the only safety mechanism
