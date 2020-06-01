---
CIP: None
Title: Cardano Metadata Server
Author: Michael Peyton Jones <michael.peyton-jones@iohk.io>, Polina Vinogradova <polina.vinogradova@iohk.io>
Comments-Summary: No comments yet
Comments-URI: TODO
Status: Draft
Type: Process
Created: 2020-06-01
License: CC-BY-4.0
---

# Abstract

We introduce a standard for metadata servers that can map opaque on-chain identifiers to metadata suitable for human consumption.
This will support user-interface components, and allow resolving trust issues in a moderately decentralized way.

# Motivation

On the blockchain, we tend to refer to things by hashes or other opaque identifiers. But these are not very human friendly: 
* In the case of hashes, we often want to know the preimage of the hash, such as
   * The script corresponding to an output locked by a script hash
   * The public key corresponding to a public key hash
* We want other metadata as appropriate, such as
   * A human-friendly name
   * The name of the creator, their website, an icon, etc. etc.
* We would like such metadata to be integrated into the UI of our applications
   * For example, if I’ve accepted a particular name for a currency, I’d like to see that name everywhere in the UI instead of the hash
* We want the security model of such metadata to be sound
   * For example, we don’t want users to be phished by misleading metadata

We think there is a case for a metadata distribution system that would fill these needs in a consistent fashion. 
This would be very useful for Plutus, multi-asset support, and perhaps even some of the existing Cardano infrastructure.
Moreover, since much of the metadata which we want to store is not determined by the blockchain, we propose a system that is independent of the blockchain, and relies on non-blockchain trust mechanisms.

The Rationale section provides additional justifications for the design decisions in this document.

## Use cases

### Script hashes

In the Goguen era of Cardano, we will see script hashes in a number of places:
* Locking outputs
* As forging policy identifiers 

In both cases it is likely that users will want to know what the script with that hash is. 
This information may be on the chain, but typically only the hash will be present until the script is actually run, for example by spending a script-locked output.
We may also want to provide other metadata, such as:

* “Higher level” forms of the code, such as the Plutus IR
* Creator information, such as contact details
* Human-readable names

Forging policy hashes are likely to be particularly in need of names: nobody wants to look at what tokens they have and see a bundle of incomprehensible hashes!

### Datum hashes

Datums in the Extended UTXO model are provided by hash, and the spending party must provide the full value. 
This is inconvenient, since the spending party needs to find out what the datum is.

If the metadata server is quick enough to register new entries, it might provide a convenient off-chain channel for this communication.

### Public keys

Communication via public keys has always faced the problem that people want to see names rather than public keys, so some form of address book is necessary.
The design proposed here is exactly like typical “address book” systems for user contact details, e.g. keysevers for PGP keys. It would therefore function as a decentralized address book for wallets.

### Public key hashes

Outputs locked by public keys are, in Cardano, actually locked by the hash of the public key. 
In some cases, users may want to make it known what key corresponds to such a hash.

### Stable addresses for oracle data 

If we choose to store things like exchange rates or credentials on-chain, for example in UTxO datums, we will want a place to put the addresses of these things. 
The fact that an output contains semantically meaningful data is metadata about that output and as such could be managed by a metadata server.

### Distributed exchange address listing

Users offering tokens for sale and exchange can lock them in contracts that say roughly "you can spend this UTxO if you send x amount of tokens to y address". 
The fact that an output constitutes an "offer" is again metadata about the output, and could be managed by a metadata server.

### Stakepool metadata

At the moment, there is a special system for distributing stakepool metadata (SMASH). 
This could potentially be replaced by the design in this document, although we would not propose this until it has been thoroughly tested out.
An alternative would be to use the existing system for stakepool metadata for the other kinds of metadata. 
However, we have decided to pursue these as parallel systems for now, because:

* The stake pool metadata system does have to monitor the chain, since the stakepool metadata is fetched from URLs which are posted to the chain.
* The stake pool metadata system is “pull-based”: it must monitor a large number of stakepool metadata URLs for updates. The metadata server system described here is “push-based”: metadata providers must push updates to the server.
* The implementation cost for a metadata server is not extremely high, as it mostly consists of a database with a small HTTP API.
* The stakepool metadata has different restrictions on content, e.g. the size limit of stakepool MD is much smaller than what we would reasonably limit script size by

# Specification

This section gives a draft specification of some parts of the system. As we will see, a virtue of this proposal is that it leaves many aspects un-specified and hence open to iterative improvement. Several features which we will discuss are therefore optional, but likely desirable in a fully-fledged implementation.

## Hash function

There are several places in this document where we need an arbitrary hash function. We will  henceforth refer to this simply as “hash”. The hash function MUST be Blake2b-256.

The hash of a string is the hash of the bytes of the string according to its encoding.

## Metadata structure

Metadata consists of a number of *properties* which describe a *metadata subject*. 
Properties consist of a mapping from *property names* to *property values*.

Metadata subjects, property names, and property values must all be represented as UTF-8 encoded strings. 
In addition, property values must parse as valid JSON.

There is no particular interpretation attached to a metadata subject: it can be anything. 
However, for the primary usecases it will be something that appears on the blockchain, like the hash of a script.

Properties with particular semantic interpretations are called “verifiable” if it is possible to check that the provided property value is correct (see below for more discussion).

We will refer to a particular property assignment for a particular metadata subject as a *metadata entry*.

### Sequence numbers

Metadata entries MUST have a sequence number associated with them. 
This is a monotonically increasing integer which is used to ensure that clients and servers do not revert to “earlier” versions of an entry.

### Attestation signatures

Metadata entries MAY have *attestation signatures* associated with them. 

Attestation signatures are annotated. 
An annotated signature for a message is a pair of a public key, and a signature of the message by the corresponding private key. 
Normal cryptographic signatures do not include the key, so the recipient must already know which key to verify it against. 
When we say “signature” in the rest of this document we mean “annotated signature”.

An attestation signature for an entry (subject, property_name, property_value, sequence_number) is a signature of the entry message:

    hash(hash(subject) + hash(property_name) + hash(property_value)  + hash(sequence_number)) [j][k][l][m][n][o][p][q]

### Examples

The following are some examples of different metadata entries for the use cases above, given as a table.


| Metadata subject         | Property name     | Property value                                                   | Sequence number | Attestation signatures                                                     |
| ------------------------ | ----------------- | ---------------------------------------------------------------- | --------------- | -------------------------------------------------------------------------- |
| `sha256(<script bytes>)` | preimage          | `{hashFn: sha256, preimage: <script bytes>}`                     |               0 | N/A                                                                        |
| `sha256(<script bytes>)` | name              | `“Fred’s script”`                                                |               3 | `(<pubkey1>, <signature by pubkey1>)`                                      |
| `sha256(<pubkey1>)`      | preimage          | `{hashFn: sha256, preimage: <pubkey1>}`                          |               4 | N/A                                                                        |
| `sha256(<pubkey1>)`      | name              | `“Fred’s key”`                                                   |               1 | `(<pubkey1>, <signature by pubkey1>), (<pubkey2>, <signature by pubkey2>)` |
| `<utxo1>`                | exchange offer    | `{fromCur: Ada, fromAmount: 5, toCur: FredTokens, toAmount: 20}` |               7 | `(<pubkey1>, <signature by pubkey1>)`                                      |
| `“gold_price_usd”`       | on-chain location | `<utxo2>`                                                        |              11 | `(<pubkey1>, <signature by pubkey1>)`                                      |
	
* Row 1 shows a script hash preimage entry, which doesn’t require a signature
* Row 2 shows a script hash name entry
* Row 3 shows a pubkey hash preimage entry
* Row 4 shows a pubkey hash name entry, signed by two keys (one of which is the key in question)
* Row 5 shows a distributed exchange offer, using a non-well-known property name, and a structured JSON object for the value
* Row 6 shows a stable address pointing to the UTXO containing the current value of the gold-to-USD exchange rate.

### Well-known properties

The following properties are considered well-known, and the JSON in their values MUST have the given structure, semantic interpretation, and verifiability. 
New properties can be added to this list by amending this CIP.

TODO: give the structures using JSON schema since values are now JSON

| Property name | Property value type                 | Interpretation                                                                                                                               | Verifiable?                                         |
| ------------- | ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| name          | string                              | A human-readable name for the metadata subject, suitable for use in an interface or in running text.                                         | No                                                  |
| description   | string                              | A longer description of the metadata subject, suitable for use when inspecting the metadata subject itself                                   | No                                                  |
| icon          | bytes                               | An icon to be used to represent the subject. Encoded as a PNG image, maximum size 64kb.                                                      | No                                                  |
| preimage      | `{hashFn: string, preimage: bytes}` | A hash function identifier and a bytestring, such that the bytestring is the preimage of the metadata subject under that hash function       | Yes, by hashing the metadata subject as instructed. |
	

## Metadata server

The metadata server exposes the functionality of a simple key-value store, where the keys are metadata subjects and property names, and the values are their property values. 

### Querying metadata

The metadata server MUST implement the following HTTP methods:

```
GET /metadata/<subject>/property/<property name>
```

This will return the property value for the given property name (if any) associated with the subject. 
This is returned as a single-entry JSON object whose key is the property name.

The metadata server MUST either:

* Use JOSE[1] to return the signatures along with the data
* Return a JSON object containing both the value and the signatures:  { “value”: value, “sequence_number”: seqeunce_number, “signatures”: signatures } (TODO: JSON Schema)

The metadata server SHOULD set the Content-Length header to allow clients to decide if they wish to download sizeable metadata.

TODO: The response type should be standardized to some structured data format with signatures, perhaps JOSE or COSE[2].

```
GET /metadata/<subject>/properties
```

This will return all the properties which are available for that subject (if any). 
These are returned as a JSON list of strings. 

Metadata servers MAY provide other methods of querying metadata, such as:

* Searching for all mappings whose “name” property value is a particular string
* Searching for all metadata items which are signed by a particular cryptographic key, or uploaded by a particular user

Servers should provide these only if they can be implemented efficiently, for example by an index in their backing database.

```
POST /metadata/query
REQUEST BODY : a JSON object with the keys:
  “subjects” : A list of subjects, encoded as strings in the same way as the queries above.
  “propertyNames” : A optional list of property names, encoded as strings in the same way as the query above.
```

This endpoint provides a way to batch queries, making several requests of the server with only one HTTP request.

If only “subjects” is supplied, this query will return a list of subjects with all their properties. 
The response format will be as similar as possible to the “GET /metadata/<subject>” request, but nested inside a list.

If “subjects” and “propertyNames” are supplied, the query will return a list of subjects, with their properties narrowed down to only those specified by “propertyNames”. 
For example, this would be an efficient way of selecting just the names and preimages for a number of subjects. 
The response format will be as before, but with fewer properties. 
Missing subjects and missing properties are ignored, leaving the response as potentially just an empty list.

### Modifying metadata

The metadata server needs some way to add and modify metadata entries. 
The method for doing so is largely up to the implementor.

1. The server MUST only accept updates for metadata entries that have a higher sequence number than the previous entry.
2. The server SHOULD require that all non-verifiable metadata entries have at least one attestation signature.
3. The server MUST always verify verifiable metadata entries before accepting them.
4. The server MUST always cryptographically verify any attestation signatures on metadata entries before accepting them.
5. The server MAY support the POST, PUT, and DELETE verbs to modify metadata entries.
6. The server MAY allow modifications via a web frontend. For example, a server could provide a full website with accounts and GUI-driven management of metadata entries.

### Authentication

If the server supports modifications to metadata entries, it SHOULD provide some form of authentication which controls who can modify them.

Servers MUST NOT use the attestation signatures on metadata entries as part of authentication. 
Attestation signatures are per-entry, and are orthogonal to determining who controls the metadata for a subject

Simple systems may want to use little more than cryptographic signatures, but more sophisticated systems could have registered user accounts and control access in that way.

#### Example authentication system: owners

An example of an authentication system that a server could use is a simple “ownership” scheme where each metadata subject is associated with an owner.

Assignment of owners may be done on a first-come-first-served basis, or using some other system (administrator action; a separate account system, etc.).

An owner is a public key. 
Updates to the properties of the metadata subject must be accompanied by an ownership signature by the owner. 
Alternatively, a multi-signature scheme could be used to allow multiple owners.

The server may choose to expose the owner as a metadata property.

An ownership signature for a subject is a signature of the message: 

```
hash(
  hash(subject) 
  + 
  hash(property_name_1) + hash(property_value_1) + hash(sequence_number_1) + hash(attestation_sig_1_1) + … + hash(attestation_signature_1_k1) 
  + 
  …
  +
  hash(property_name_n) + hash(property_value_n) + hash(sequence_number_n) + hash(attestation_sig_n_1) + … + hash(attestation_sig_n_kn))
```


Where `property_name_1 … property_name_n` are the property names, ordered lexicographically, and `attestation_sig_1 … attestation_sig_kn` are the attestation signatures for the nth metadata entry, ordered lexicographically by the attesting key.

TODO: is this precise enough?

### Auditability

The metadata server MAY provide a mechanism auditing changes to metadata, for example by storing the update history of an entry and allowing users to query it (via the API or otherwise). 

## Metadata client

The metadata client refers to the component that communicates with the metadata server and maintains the user’s trusted metadata mapping. 
This may be implemented as part of a larger system, or may be an independent component (which could be shared between multiple other systems, e.g. a wallet frontend and a blockchain explorer).

### Server configurability

The metadata client MUST allow the metadata server (or servers) that it references to be configured by the user.

### Metadata storage

The metadata client MUST maintain a local mapping of trusted metadata entries.

The metadata client SHOULD allow browsing of its local mapping, as well as modifications by the user.

The metadata client SHOULD use a Trust On First Use (TOFU) strategy for updating this store. 
When an update is requested for a metadata entry, the client should query the server, but not trust the resulting metadata entry unless the user explicitly agrees to trust it. 
If the entry is trusted, it should be copied to the local store.

The metadata client SHOULD expose the following interfaces for metadata consumers:

1. Get the metadata entry for a particular subject and property name from the local storage, if it exists.
2. Update the metadata entry for a particular subject and property name by querying the client’s configured metadata server (or servers), querying the user for acceptance as described above.

The metadata client MAY also provide bulk update operations.

The metadata client MAY be configurable to limit the amount downloaded.

#### Trust

The metadata client MUST only accept updates with a higher sequence number than the entry it currently has.

The metadata client MUST always verify verifiable metadata entries before accepting them.

The metadata client MUST always verify any signatures on metadata entries before accepting them.

The metadata client MAY accept updates for verifiable metadata entries without user action.
 
## Metadata consumer

The metadata consumer is a user-facing application that wishes to make use of metadata, for example a wallet frontend or a blockchain explorer. 
A metadata consumer may incorporate a metadata client.

### Trust

The metadata consumer MUST relay any prompts for user trust that come from the metadata client.

The metadata consumer MAY allow the user to trust a cryptographic key, such that updates to entries which are signed by that key will be accepted without user action in the future. 
This may have limited scope, for example the user might trust it globally or only with respect to a specific metadata subject or subjects.

### Server configurability

The metadata consumer MUST expose the ability to configure the server that the metadata client uses.

### User modifications

The metadata consumer SHOULD expose the ability for users to modify local metadata entries, for example changing a piece of “name” metadata to avoid a conflict.

### Consistency

The metadata consumer MAY require some additional consistency checks before it accepts updates, for example it may require that no two metadata subjects for currencies have the same “name” metadata. 
Such conflicts MUST be reported to the user for resolution.

## Metadata submitter

A metadata submitter is a user or application that wishes to submit records to a metadata server for inclusion, so that consumers will receive them.

The exact interface for metadata submitters is not currently specified in this document and may vary by server implementation. 
The following sections are provided as an indication of a possible approach, but are not normative.

### Metadata generation tools

A server implementation SHOULD come with tools for metadata submitters to generate and submit records to the server.

This should include:
* Credential management for attestation signatures
* Tools for generating messages and signing them with credentials

#### Credential management

To sign metadata entries (attestation and owner signatures) the tool needs to generate a set of keys exclusively for this purpose. 
These keys will provide the necessary "credentials" for any further attestations (attestation signatures) or any modifications to existing entries in a given registry (owner signatures).

A basic implementation MAY use a 32byte ed25519 key pair. The tools may support generating and managing these, or the user must be able to handle key management themselves.

Any tools provided for generating metadata entries and signing them will need access to these key files. For a basic implementation of such tools we therefore propose using a cli application.

#### Attestation signatures

The tool must be able to generate an attestation signature for a metadata entry, when provided with the following input:
* subject (string)
* property_name (string)
* property_value (string)
* signing key (file)

Any application should derive the public key from the signing key and output a JSON representation of the annotated (attestation) signature with public key and signature as hex strings.

#### Owner signatures

The tool must be able to generate an owner signature for a metadata record, when provided with the following information as input:
* subject (string)
* all property_names, property_values and annotated signatures for the record (JSON, file)
* signing key (file)
Any application should derive the public key from the signing key and output a JSON representation of the annotated (owner) signature with public key and signature as hex strings.

#### Sequence numbers

The application needs to include a new sequence number for the entry, but this can simply be passed as an argument by the user.

### Basic implementation hints
A very basic cli application could provide the functionality described above and require the user to manually generate and append annotated signatures iteratively for each metadata property. A complete JSON representation of the record could then be used as input to generate the annotated (owner) signature in a second step.

Other implementations could provide this functionality in one step, but much of the complexity lies in the variation of possible user input, since a record can be composed of multiple metadata properties, with multiple attestations by different keys[v]. 

# Rationale

## Interfaces and progressive enhancement

The design space for a metadata server is quite large. For example, any of the following examples could work, or other combinations of these features:

* A single centralized server maintained by the Cardano Foundation, and updated by emailing the administrator (no user-facing front end website)
* An open-source federation of servers with key-based authentication.
* A commercial server with an account-based system, a web-frontend, and a payments system for storage.

This design aims to be agnostic about the details of the implementation, by specifying only a very simple interface between the client, server, and system-component users (wallet, etc.). This allows:

* Progressive enhancement of a IOHK- or Cardano Foundation-provided server over time. Initially we may provide a very bare-bones implementation, but this can be improved later.
* Competition between multiple implementations, including from third parties.

## Decentralization

For much of the metadata we are concerned with (i.e. non-verifiable metadata), there is no “right” answer. 
The metadata server is thus playing a key trusted role - even if that trust is partial because users can rely on attestation signatures. 
We therefore believe that it is critical that we allow users to choose which metadata server (or servers) they refer to. 

An analogy is with DNS nameservers: these are a trusted piece of infrastructure for any particular user, but users have a choice of which nameservers to use, and can use multiple.

This also makes it possible for these servers to be a true piece of community infrastructure, rather than something wedded to a major player (although we hope that the Cardano Foundation and IOHK will produce a competitive offering).

## Verifiable vs non-verifiable metadata

A key distinction is between metadata that is verifiably correct and that which is non-verifiable.

The key example of verifiable metadata is hashe pre-images. 
Where the metadata subject is a hash, the preimage of that hash is verifiable metadata, since we can simply hash it and check that it matches. 
This covers several cases that we are interested in:

* Mapping script hashes to the script
* Mapping public key hashes to the public key

However, most metadata is non-verifiable:

* Human-readable names for scripts are not verifiable: there is no “right” answer to what the name of a script is.
   * Many scripts do not have an obvious “owner” (certainly this is true for Plutus scripts), but even for those which do (e.g. multisig scripts) this does not mean that the owner is trusted! See the “Security” section below for more discussion on trust.
* Any associations with authors, websites, icons, etc are similarly non-verifiable as there is no basis for establishing trust.

## Security 

Most of the security considerations relate to non-verifiable metadata. 
Verifiable metadata can generally always be accepted as is (provided that it is verified). 
Our threat model is that non-verifiable metadata may always have been provided by an attacker.

### Server security

Allowing unrestricted updates to non-verifiable metadata would allow malicious users to “squat” or take over another user’s metadata subjects. 
This is why we recommend that server implementations have some kind of authentication system to mitigate this.

One approach is to have a system of “ownership” of metadata subjects. 
This notion of ownership is vague, although in some cases there are obvious choices (e.g. for a multisig minting policy, the obvious owners are the signatories who can authorize minting). 
It is up to the server to pick a policy for how to decide on owners, and enforce security; or indeed to take a different approach entirely.

See the earlier “Authentication” section for a description of a possible approach to managing ownership.

Depending on the authentication mechanism they use, servers may also need to worry about replay attacks, where an attacker submits a (correctly signed) old record to “revert” a legitimate change. 
However, these can be unconditionally prevented by correctly implementing sequence numbers as described earlier, which prevents old entries from being accepted.

### Client security

Accepting non-verifiable metadata blindly can lead to attacks. 
For example, a malicious server or user might attempt to name their currency “Ada”. 
If we blindly accept this and overwrite the existing mapping for “Ada”, this would lead to easy phishing attacks.

The approach we take is heavily inspired by petname systems, GPG keyservers, and local address books. 
The user always has a local mapping which is trusted, and then adding to or updating that mapping requires explicit user consent, unless we can prove that this is trustworthy (i.e. the metadata is verifiable).

How can users decide whether to trust an update? 
This is where the attestation signatures come into play. 
If a user trusts the entity which signs the metadata record, that may be sufficient for them to accept it as a legitimate update. 
See the earlier “Trust” sections for some discussion of trusted keys and updates.

Clients could also be tricked into downloading large amounts of metadata that the user does not want. 
For this reason clients should expose some kind of configurable download limiting, and we suggest that the server set the Content-Length header to support this. 
However, this problem is no worse than that faced by the average web browser, so we do not think it will be a problem in practice.

Clients also need to worry about replay attacks, where they are sent old (correctly signed) records in an attempt to “roll back” a legitimate update. 
The easiest way to avoid this is to correctly implement sequence numbers, in which case old updates will be rejected.

## Client/consumer split

This document features a perhaps surprisingly large number of components. 
Some of these are uncontroversial: there must be a server and a client, and you need a tool to do submission.

But the client/consumer split is less clear. Strictly, it is not necessary, and the two could be treated as one component. 
But splitting them allows the client to be a simpler, self-contained component, while the consumer may have concerns that reach into the details of the application that wants to use the metadata.

## Consumer sanity checking

Keeping most of the sanity checking (e.g. that names are unique) in the consumer simplifies the behaviour of the client and server significantly, and allows different consumers to adopt different policies as appropriate, even if they share a client.

## Data storage location

This design needs to store a fair amount of data in a shared location. 
We might wonder whether we should use the blockchain for this: we could store metadata updates in transaction metadata.

However, storing this information on-chain does not actually help us: 

* The trust model for metadata is different than that of the ledger and transactions. The only trust we have (and can expect to have) in the metadata is that it is signed by a particular key, regardless of the purpose or nature of the data.
   * E.g. when posting a script, there is no explicit association between the script and the signing key other than the owner of the key choosing to post it
* The metadata is precisely that: metadata. While it is about the chain, it does not directly affect ledger state transitions and therefore we should not require it to be associated with a specific transaction.

On the other hand, on-chain storage comes with considerable downsides:

* Higher cost to users for modifications and storage
* Increases in the UTXO size
* Awkwardness of querying the data
* Size limits on transaction metadata

For this reason, we think that a traditional database is a much better fit. 
However, it would be perfectly possible for someone to produce an implementation that was backed by the chain if they believed that that could be competitive.

## Storage cost

Metadata may potentially be sizable. For example, preimages of hashes can in principle be any size!

Servers will want some way to manage this to avoid abuse. However, this is a typical problem faced by web services and can be solved in the usual ways: size limits per account, charging for storage, etc.

# References

[1] https://jose.readthedocs.io/en/latest/
[2] https://tools.ietf.org/html/rfc8152
