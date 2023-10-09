---
title: "Interoperable Private Identity Discovery for E2EE Messaging"
abbrev: "E2EE Messaging Private User Discovery"
category: info

docname: draft-party-mimi-user-private-discovery-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: ART
workgroup: "More Instant Messaging Interoperability (mimi)"
keyword:
 - interoperable E2EE messaging
 - private user discovery
venue:
  group: mimi
  type: Working Group
  mail: mimi@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/mimi/
  github: femigolu/giles-interop-user-private-discovery
  latest: https://datatracker.ietf.org/doc/giles-interop-user-private-discovery/

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: Giles Hogben
    organization: Google
    email: gih@google.com
 -
    fullname: Femi Olumofin
    organization: Google
    email: fgolu@google.com

normative:
  PIRFramework:
    author:
      - fullname: Sarvar Patel
      - fullname: Joon Young Seo
      - fullname: Kevin Yeo
    title: >
      Don't be Dense: Efficient Keyword PIR for Sparse Databases
    seriesinfo: 32nd USENIX Security Symposium, USENIX Security 2023

informative:


--- abstract

This document specifies how users can find each other privately when using end-to-end encrypted messaging services. Users can retrieve the key materials and message delivery endpoints of other users without revealing their social graphs to the key material service hosts. Users can search for phone numbers or user IDs, either individually or in batches, using private information retrieval (PIR). Our specification is based on a state-of-the-art lattice-based homomorphic PIR scheme, which provides a reasonable tradeoff between privacy and cost in a keyword-based sparse PIR setting.


--- middle

# Terminology

{::boilerplate bcp14-tagged}

Glossary of terms:

- Authoritative Name Server: Final holder of the IP addresses for a specific domain or set of domains.
- Client: A software application running on a user's device or computer.
- Database: A collection of records all of equal sizes (i.e., padded as appropriate).
- Dense PIR: A type of PIR scheme that is used to retrieve information from a database using the index or position of each record as key. This is equivalent to the standard PIR schemes from the literature.
- DNS: See Domain Name Service.
- ​​Domain Name Service: A system that accepts domain names and returns their IP addresses.
- FHE: See Fully Homomorphic Encryption.
- Fully Homomorphic Encryption: A type of encryption that allows arithmetic operations to be performed on encrypted data without decrypting it first.
- KDS Resolver: A service that helps clients find and download the public keys of other users.
- KDS: See Key Distribution Server.
- Key Distribution Server: A server holding the public key material that enables a user to securely communicate with other users.
- Name Server: Stores DNS records that map a domain name to an IP address.
- Partition: A smaller division of a shard that is used to facilitate recursion with PIR.
- PIR: See Private Information Retrieval
- Preferred Service: A messaging service that a user has chosen as the default.
- Private Information Retrieval: A cryptographic technique that allows a client to query a database server without the server being able to learn anything about the query or the record retrieved.
- Public Key Bundle: Cryptographic key and other metadata that are used to encrypt and decrypt messages.
- Public Key PIR: A type of PIR scheme that uses a small amount of client storage to gain communication and computation efficiencies over multiple queries.
- Resolver: A service that helps clients find the IP address for a domain through recursive queries over Name Servers hierarchy.
- Shard: A subset of a large database that is divided into smaller, more manageable pieces.
- Sparse PIR: A type of PIR scheme that is used to retrieve information from a database of key-value pairs. This is the same as Keyword PIR in the literature.
- Transform: A process of converting the partitions in a shard into a format that is suitable for homomorphic encryption computations.

# Introduction

Outline of design for message delivery bridge and key distribution server discovery mechanisms for interoperable E2EE messaging clients. A DNS-like resolver service stores UserID <-> service pairs that map to key distribution and message delivery endpoints (e.g. Platform1 Bridges).  Each service is responsible for an "authoritative name server" that covers its own users and this can be replicated/cached by other providers and retrieved by sender clients using a privacy preserving protocol to prevent social graph leakage.

## Functional Requirements

For a given messaging service identity handle (phone number or alphanumeric UserID):

1. **Endpoint discovery**: discover receiver service IDs to retrieve public key material and send message payload e.g. [Platform1.org/send/](matrix.org/send/), [Platform1.org/kds](matrix.org/kds) 

2. **Default service discovery**: Discover optional default receiver service ID user preference for a given PN/UserID (e.g. default:Platform1.org)

3. **Global uniqueness**: Fully-qualified service identifiers should be globally unique

4. **Key verification (P1)**: Provide an independently trusted party to assert and verify the association between a public key and a UserID

## Privacy Requirements

1. **Social graph**: Resolver or discovery services should not learn the PN/UserID a client is querying for (i.e. who is sending a message to who)

2. **Querying user identity**: A resolver service should not default to sharing the querying user identity with other resolver services when it requires their help for discovery

3. **Metadata**: Resolver service should not learn the exact timing of when a message is sent

## Other Non-functional Requirements

1. No single entity should be financially responsible for resolving all identity queries (e.g. even within a geographical region)

2. Costs for each participating entity of storing and querying key records should be proportional to their number of participating users

3. Performance should support each client querying each of their contacts at least once every 24 hours

## Non-requirements

1. **Hiding service reachability**: This is the link between a UserID and a service. All major E2EE messaging services already publish unACL'd reachability information without opt-out i.e. +16501234567, reachable on Messages, Whatsapp, Telegram (not including name or any other info). Therefore this should not be a goal (and would not be feasible to implement).

2. **Hiding the value of UserIDs or public keys**: e.g. the existence of the PN, +16501234567

3. **Hiding the association between a public key and a UserID**: e.g. PN +16501234567 has pubkey x

4. **Contact lookup by name** (or anything except username)

# Proposed solution


## Key distribution

~~~ plantuml-utxt
participant "P1\n Client" as A
participant "P1\n Front End" as B
participant "P1\n Name Server" as C
participant "Authoritative P2\nName Server" as D
participant "P2\n KDS" as E
participant "P2\nClient" as F

C->D: Request P2\nName Records
D->C: Replicate P2\nName Records
A->B: PIR Query\nPN/UserID
B->C: PIR Query\nPN/UserID
note over C, B
  Supported service IDs
   + default service
end note
C->B: Service IDs\n& default service
B->E: Query\nPN/UserID
E->B: Return Public\nKey Bundle
B->A: Return Public\nKey Bundle
A->A: Encrypt Message
A->F: Send Encrypted Message via messaging providers
~~~

Taking Platform1 client sending to a Platform2 user as an example:


1. Platform1 name server replicates authoritative Platform2 NS records. There will need to be a shared list of participating services and name server endpoints.

2. Platform1 client sends key bundle request to Platform1 front end (PIR encrypted PN/UserID)

3. Platform1 FE gets supported key distribution service IDs, version number + default service=Platform2 via PIR protocol from its own name server.

4. Platform1 FE queries Platform2 KDS to retrieve public keys.

  *   4.1 Platform1 Client first sends (query and session key) encrypted with Platform2 public key to Platform1 FE.
  *   4.2 Platform1 FE sends encrypted query to Platform2 KDS
  *   4.3 Platform2 KDS decrypts query and session key, encrypts response with session key
  *   4.4 Platform2 KDS sends encrypted response to Platform1 FE
  *   4.5 Platform1 FE forwards to Platform1 client

{:start="5"}
5. Platform 1 Client and Platform 2 Client exchange messages through their respective messaging providers.

This provides E2EE interop while only disclosing to gateway service which services a phone number is registered to. In all other respects, gateway services learn no more information than in the non-interop case.

## Resolver registration

Each service is responsible for registering user enrollments with the resolver.


## Preferred service integrity

While the preferred service is public, the user should control its value/integrity. As well as ensuring user control, it also prevents spoofing attacks where an attacker A could create an account on a new service that B does not have an account on, and then set it to B's preferred service (see cross-service identity spoofing below). Therefore to prevent anyone but the user modifying the default service value, records must be signed with the user's private key and verified by the sender against their public key. For multiple key pairs across different services, the last key pair to sign the default service bit must be used to change the default.


~~~ plantuml-utxt
participant Client as A
participant "Service UI" as B
participant Resolver as C

B->C: Register Preferred Service + Signature
A->C: Query PN/UserID
C->A: Return supported service IDs + default service preference + signature
A->A: Verify default service pref signature
~~~


## Cross-service identity spoofing

Today, a messaging service may support one or more ways of identifying a user including email address, phone number, or service specific user name.

Messaging interoperability introduces a new problem that traditionally has been resolvable at the service level: cross-service identity spoofing, where a user on a given E2EE may or may not be addressable at the same ID on another service due to a lack of global uniqueness constraints across providers.

As a result, a user may be registered at multiple services with the same handles, e.g. if Bob's email is [bob@example.com](mailto:bob@example.com) and his phone number is 555-111-2222 and he is registered with Signal and iMessage, he would be addressable at [bob@example.com](mailto:bob@example.com):iMessage, 555-111-2222:iMessage, and 555-111-2222:Signal. In this case, the same userId on iMessage and Signal is acceptable as the phone number can map to only one individual who proves their identity by validating ownership of the SIM card.

On services where a user can log in with a username _alone_, however e.g. Threema and FooService, the challenge becomes:


*   Alice messages Bob at Bob's preferred service (bob@Threema)
*   Eve messages Alice impersonating Bob using bob@FooService
*   Alice needs some indicator or UI to know that bob@Threema isn't bob@FooSercice and that when bob@FooService messages, it should not be assumed that bob@FooService is bob@Threema.

Options for solving this are:
1. Storing the supported services for a contact in Contacts and if a receipt receives a message from an unknown sender, to treat it as spam or otherwise untrusted from the start.
2. Requiring the fully qualified username for services that rely on usernames only - e.g. bob@threema.com vs bob.

# Privacy of resolver lookup queries

Resolver lookup queries leak the user's social graph - i.e. who is communicating with whom, since the IP address of the querying client can be tied to user identity, especially when operating over a mobile data network. Therefore we propose to use Private Information Retrieval (PIR) to perform the resolver queries. We have evaluated multiple alternative schemes. The proposed solution is based on the Public Key PIR framework by Patel et al{{PIRFramework}} with sharded databases. This framework is applicable with any standard PIR scheme such as the open source implementation [here](https://github.com/google/private-retrieval). Cost estimates suggest this is feasible even for very large resolver databases (10 billion records).

## Proposed protocols

A private information retrieval protocol enables a client holding an index (or keyword) to retrieve the database record corresponding to that index from a remote server. PIR schemes have communication complexities sublinear in the database size and they provide access privacy for clients which precludes the server from being able to learn any information about either the query index or the record retrieved. A standard single-server PIR scheme provides clients with algorithms to generate a query and decrypt a response from the server. It also provides an algorithm for the server to compute a response.

The Public Key PIR framework {{PIRFramework}} can be wrapped around any standard lattice-based PIR scheme. This framework consists of server database setup, client key initialization, client query generation, server response computation, and client response decryption sub-protocols. All operations are over a set of integers with a plaintext modulus. The appendix provides the protocol messages in protocol buffer format.


### Server database setup



*   **Sharding**: If the database is over 2<sup>20</sup> records, sub-divide it into  shards of ~1 million unique records each, which is a good balance for privacy and costs. Performing PIR over the databases gives stronger privacy but is more costly. Similarly, running PIR queries over the shards is less costly but gives weaker privacy.
    *   Sample a hash key **K<sub>s</sub>** for sharding.
    *   Using **K<sub>s</sub>**, shard the large database of **r** records into **&rceil;r/2<sup>20</sup>&lceil;** shards based on the hash prefix of the record's unique identifier.
    *   **N.B.** The hash key will be provided to clients to determine the shard to query.
*   **Set partitioning boundaries for each shard D**: Given a **n** key-value pairs shard **D = {(k<sub>1</sub>,v<sub>1</sub>),...,(k<sub>n</sub>,v<sub>n</sub>)}**, then
    *   Compute the number of database partitions as **b = n/d<sub>1</sub>**. Where **d<sub>1</sub>** is the desired size for each partition. A value of 128 for **d<sub>1</sub>** works well.
    *   Sample a partitioning boundary hash key **K<sub>1 </sub>** to generate a target partition for each shard key.
    *   Compute the hash **F<sub>1</sub>(K<sub>1</sub>,k<sub>i</sub>)** for each record identifier **k<sub>i</sub>**.
    *   Sort the hash values alphanumerically and then divide the list into **b** partitions **P<sub>1</sub>,...,P<sub>b</sub>**.
    *   Store the b partition boundaries beginning hash values **B<sub>0</sub>, ..., B<sub>b</sub>**. Note that **B<sub>0</sub> = 0**, and **B<sub>b</sub> = \|U\|-1** where U is the rage for **F<sub>1</sub>(K<sub>1</sub>,k<sub>i</sub>)**.
    *   **N.B.** The partition boundaries will be provided to clients and can be stored efficiently (e.g., ~11KB for **n** = 2<sup><strong>20</strong></sup>, **d<sub>1</sub>** = 128, **\|U\|** = 2<sup><strong>64</strong></sup>).
*   **Transform each shard**: Sample two hash keys **K<sub>2</sub>** and **K<sub>r</sub>** where **K<sub>2</sub>** will be used to generate a row vector within each partition, and **K<sub>r</sub>** is used to generate a representation for the transformed database as **F(K<sub>r</sub>,k<sub>i</sub>)\|\|v**.
*   **N.B.** **F(K,k)** is the output value from hashing **k** with key **K** and **\|\|** is a concatenation.
*   For each partition **P<sub>i</sub>**
    *   Construct a **\|P<sub>i</sub>\| x d<sub>1</sub>** Platform1 **M<sub>i</sub>** by appending a random row vector from the bit vector derived from **(F<sub>2</sub>(K<sub>2</sub>,k\|\|1),...,F<sub>2</sub>(K<sub>2</sub>,k\|\|d<sub>1</sub>))**.
    *   Construct a **\|P<sub>i</sub>\|** vector **y<sub>i</sub>** by appending **F<sub>r</sub>(K<sub>r</sub>,k)\|\|v** for each **(k,v)** in **P<sub>i</sub>**.
    *   Solve for **e<sub>i</sub>** that satisfies **M<sub>i</sub>e<sub>i</sub> = y<sub>i</sub>**.
*   Construct the transformed **d<sub>1</sub> x b** Platform1 as **E = [e<sub>1</sub> … e<sub>b</sub>]**.
*   The Platform1 **E** is the transformed Platform1 for shard **D**.
*   The clients must download parameters **(K<sub>1</sub>,K<sub>2</sub>,K<sub>r</sub>)** to query each shard, plus **K<sub>s</sub>** to inform the server of the target shard for a query.

This protocol is completed by the server without any client participation and before answering any client query. Note that a shard must be re-transformed after an update. Shard transform only takes a few minutes.


### Client key initialization



*   The client generates a per-key unique identifier (**UID**),  **private key** and **public key** using a fully homomorphic encryption (FHE) scheme relying on hardness assumptions similar to Ring Learning with Errors problem.
*   The client persists the **UID** and **private key** into its local key store, and uploads query-independent parameters **UID** and **public key** to the server. These later parameters will enable the server to perform cryptographic operations (i.e., FHE) efficiently.
*   The server needs to maintain an up-to-date mapping of **UID** to **public key** for all clients.
*   Each client completes this offline initialization protocol before running any query. It also needs to perform it periodically (e.g., weekly or monthly) to prevent server linkability of private queries to the user over an extended period.
*   The client downloads query parameters from the server:
    *   Sharding hash key **K<sub>s</sub>** to inform the server of the target shard for a query.
*   Sets of parameters (**K<sub>1</sub>,K<sub>2</sub>,K<sub>r</sub>,B<sub>0</sub>, …, B<sub>b</sub>**) for each shard.

### Client query generation



*   The client creates a query to retrieve the value corresponding to a database key **k** as follows:
*   Select a standard PIR algorithm with server-supported implementation as the underlying PIR scheme.
*   Compute **d = F<sub>s</sub>(K<sub>s</sub>,k)** to identify the shard to query.
*   Compute **j = F<sub>1</sub>(K<sub>1</sub>,k)** to learn which partition contains the desired entry from the downloaded partition boundaries for the shard.
*   Generate **z** vector **v** of length **d<sub>1</sub> , … , d<sub>z</sub>** . Compute a **d<sub>1</sub>**-length random bit vector **v<sub>1</sub>** from **(F<sub>2</sub>(K<sub>2</sub>,k\|\|1),...,F<sub>2</sub>(K<sub>2</sub>,k\|\|d<sub>1</sub>))**. Compute **v<sub>2</sub>** as a zero bit vector of **d<sub>2</sub>** length with only the bit set at **&lfloor;j/&rceil;n/d<sub>1</sub>d<sub>2</sub>&lceil;&rfloor;**. Similarly compute **v<sub>3</sub> , … , v<sub>z</sub>**.
*   Finally use the underlying PIR scheme and the private key to encrypt the **z** vector **v.**
*   Send **v, d** and the **UID** to the server.
*   **N.B.** The dimension **d<sub>z</sub>** is typically small; a size of 2 or 4 works well.

### Server response computation



*   The server retrieves the public key for the client's **UID**, and computes the ciphertext of the value corresponding to the key of interest for the shard **d**, as follows.
*   Take the transformed shard **E** as a **d<sub>1 </sub>x &rceil;n/d<sub>1</sub>&lceil;** Platform1 **E<sub>1</sub>**, use the underlying PIR response answering algorithm to compute **v<sub>1</sub>.E<sub>1</sub>**, and rearrange the resulting **&rceil;n/d<sub>1</sub>&lceil;** vector as a **d<sub>2 </sub>x &rceil;n/d<sub>1</sub>d<sub>2</sub>&lceil;** Platform1 **E<sub>2</sub>**.
*   Next, compute **v<sub>2</sub>.E<sub>2</sub>**, and rearrange the resulting **&rceil;n/d<sub>1</sub>d<sub>2</sub>&lceil;** vector as a **d<sub>3 </sub>x &rceil;n/d<sub>1</sub>d<sub>2</sub>d<sub>3</sub>&lceil;** Platform1 **E<sub>3</sub>**.
*   The server similarly repeats the computation for the remaining dimensions **v<sub>3</sub> ,… , v<sub>z</sub>**.
*   The end result is a ciphertext **r** of the database value corresponding to **k**. The server sends **r** back to the client.


### Client response decryption



*   The client uses the underlying PIR response decryption algorithm and **private key** to decrypt the response **r** as **k<sub>r</sub>\|\|v**. If **F<sub>r</sub>(K<sub>r</sub>,k) == k<sub>r</sub>** then returns **v** otherwise returns **null** (key not found).

## FHE key requirements


*   At least 128-bit of security
      - ElGamal, NIST P-224r1 curve and a 4 bytes plaintext size for fast decryption.
      - Gentry-Ramzan, used a 2048-bit modulus
      - Damgård-Jurik, used 1160-bit primes

## Cost estimates

In these estimates, we propose using shards of size ~1 million of identifiers. For 1.28 TB (10 billion records), breaking this down into 10,000 shards each of size 1 million records gives a cost estimate for each query as below:


| Parameter                                                | Cost estimate     |
|----------------------------------------------------------|-------------------|
| PIR Public Key Size Per Device, including metadata (storage required)    | 14 MB             |
| Upload Bandwidth Per Query                               | 14 KB             |
| Download Bandwidth Per Query                             | 21 KB             |
| Client Time Per Query                                    | 0.1s              |
| Server Time Per Query (Single Thread)                    | 0.8-1s            |



Note on some assumptions for feasibility:



1. Resolver queries will be cached (vs requiring a roundtrip for every message) and asynchronous with message sending, therefore 1s latency is acceptable.
2. It is acceptable for key changes to be communicated reactively on decrypt failure.
3. Group messaging E2EE is bootstrapped using individual users' public keys and for V1, group state will be stored by the initiating user's service provider. Therefore no additional discovery mechanism is required.


## Notes


# IANA Considerations

This document has no IANA actions.

## Appendix
The protocol messages are specified in protobuffer format in this appendix.

~~~protobuf
syntax = "proto3";

package mimi.discovery.pir;

// PirParameterRequest represents a request from a client to a server to
// fetch database and cryptographic parameters for querying a database.

message PirParameterRequest {
  // The protocol version for PIR.
  uint32 protocol_version = 1;

  // Used to prevent replays; request fingerprint.
  string nonce = 2;

  // The unique identifier of the client.
  string uid = 3;

  // The PIR scheme that the client wants to use.
  PirScheme pir_scheme = 4;

  // The public key of the client.
  repeated bytes public_key = 5;
}

// PirParameterResponse represents a server's response to PirParameterRequest
// and it contains database and cryptographic paramaters to use for quering the
// database using PIR.

message PirParameterResponse {
  // The protocol version for PIR.
  uint32 protocol_version = 1;

  // Used to prevent replays; request fingerprint.
  string nonce = 2;

  // The unique identifier of the client.
  string uid = 3;

  // Status of the request.
  PirStatus response_status = 4;

  // The PIR scheme negotiated from the client request.
  optional PirScheme pir_scheme = 5;

  // A map of server-supported PIR schemes to their paramaeters.
  map<PirScheme, PirParams> pir_schemes = 6;

  // The shard hash key to identify subdatabase cointaining a keyword.
  HashKey shard_key = 7;

  // parameters for sub databases
  repeated PirSubDbParameter sub_db_param = 8;
}

// PirSubDbParameter represents a subdatabase partitioning parameters

message PirSubDbParameter {
  // The hash key of the partition boundary.
  HashKey partion_boundary_hash_key = 1;

  // The hash key of the row vector.
  HashKey row_vector_hash_key = 2;

  // The hash key of the record representation.
  HashKey record_representation_hash_key = 3;

  // The beginning hash values for the partition
  // boundaries for a single shard.
  repeated PatitionBundaries partition_boundaries = 4;

  // The size of the database.
  uint32 database_size = 5;

  // The size of the largest record.
  uint32 record_size = 6;
}

// PirRequest represents a sparse PIR request to a server for retrieving a
// database record.

message PirRequest {
  // The protocol version for PIR.
  uint32 protocol_version = 1;

  // Used to prevent replays; request fingerprint.
  string nonce = 2;

  // The unique identifier of the client.
  string uid = 3;

  // The PIR scheme that the client wants to use.
  optional PirScheme pir_scheme = 4;

  // The shard ID of the record to be retrieved.
  uint32 shard = 5;

  // The encrypted vector of the query.
  repeated bytes encrypted_vector = 6;
}

// PirResponse represents a PIR server's response to a PirRequest.

message PirResponse {
  // The protocol version for PIR.
  uint32 protocol_version = 1;

  // Used to prevent replays; request fingerprint.
  string nonce = 2;

  // The unique identifier of the client.
  string uid = 3;

  // Status of the request.
  PirStatus response_status = 4;

  // The encrypted response.
  repeated bytes encrypted_response = 5;
}

// PatitionBundaries represents the beginning hash values for the partition
// boundaries for a single shard.

message PatitionBundaries {
  // The hash values of the partition boundaries.
  repeated string boundary = 1;
}

// HashKey represents a cryptographic hash key.

message HashKey {
  // The hash key.
  repeated bytes hash_key = 1;
}

// PirParams represents the paramaters of a PIR scheme instance.

message PirParams {
  // The number of levels of recursion used in the PIR scheme.
  optional int32 levels_of_recursion = 1;

  // The modulus used in the PIR scheme.
  repeated bytes modulus = 2;

  // The plaintext modulus used in the PIR scheme.
  repeated bytes plaintext_modulus = 3;

  // Other parameters used in the specific PIR scheme.
  repeated bytes other_params = 4;

  // Reserved values.
  reserved 5, 6, 7;
}

// PirScheme represents an indexed-based PIR scheme .

enum PirScheme {
    // Scheme not specified
    PIR_SCHEME_UNSPECIFIED = 0;

    // XPIR scheme based on the Ring Learning with Errors (Ring-LWE) problem.
    PIR_SCHEME_RLWE_XPIR = 1;

    // Add more PIR schemes that we might want to support.
}

// PirStatus represents the PIR server response codes.

enum PirStatus {
  // The PIR scheme selected by the client is supported.
  PIR_SCHEME_MATCHED = 0;

  // The PIR scheme selected by the client is not supported, but the server
  // suggests a list of supported schemes.
  PIR_SCHEME_SUGGESTED = 1;

  // There are problems with the request.
  PIR_REQUEST_FAILURE = 2;

  // The response was successfully computed from the request.
  PIR_RESPONSE_SUCCESS = 3;

  // The response computation failed.
  PIR_RESPONSE_FAILURE = 4;
}
~~~

--- back

# Acknowledgments
{:numbered="false"}

The technical description of the private information retrieval framework is based on Sarvar Patel, Joon Young Seo and Kevin Yeo's USENIX Security '23 paper titled ["Don't be Dense: Efficient Keyword PIR for Sparse Databases
"](https://www.usenix.org/conference/usenixsecurity23/presentation/patel).

