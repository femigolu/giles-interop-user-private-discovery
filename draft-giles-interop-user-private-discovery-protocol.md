---

title: "Interoperable Privacy Preserving User Identity and Discovery for E2EE Messaging"
abbrev: "Interoperable Private User Discovery for E2EE Messaging"
category: info

docname: draft-giles-interop-user-private-discovery-protocol
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
  latest: https://femigolu.github.io/giles-interop-user-private-discovery/giles-interop-user-private-discovery-protocol.html
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

informative:


--- abstract

This document specifies how users can find and communicate with each other privately when using end-to-end encryption messaging. Users can retrieve the key materials and message delivery endpoints of other users without revealing their social graphs to the key material service hosts. Users can search for phone numbers or user IDs, either individually or in batches, using private information retrieval (PIR). Our specification is based on the state-of-the-art lattice-based homomorphic PIR scheme, which provides a reasonable tradeoff between privacy and cost in a keyword-based sparse PIR setting.


--- middle

# Introduction

Outline of design for message delivery bridge and key distribution server discovery mechanisms for interoperable E2EE messaging clients. A DNS-like resolver service stores UserID <-> service pairs that map to key distribution and message delivery endpoints (e.g. Platform1 Bridges).  Each service is responsible for an "authoritative name server" that covers its own users and this can be replicated/cached by other providers and retrieved by sender clients using a privacy preserving protocol to prevent social graph leakage.


## Functional Requirements

For a given messaging service identity handle (Phone number or alphanumeric UserID):


1. P0 Return receiver service ID to send message payload: can be mapped to endpoints for message delivery and key distribution using standard mechanism -> e.g. [Platform1.org/send/](matrix.org/send/), [Platform1.org/kds](matrix.org/kds) 

2. P0 Return optional default receiver service ID user preference for a given PN/UserID (e.g. default:Platform1.org)


## Privacy Requirements

* P0 Resolver service should not learn the PN/UserID a client is querying for (i.e. who is sending a message to who)

* P0 Resolver service should not learn the public identity of the querying client.

* P0 Resolver service should not learn the exact timing of when a message is sent 

## Privacy Non-requirement

Hiding service reachability. All major E2EE messaging services already publish unACL’d reachability information without opt-out i.e. +16501234567, reachable on Messages, Whatsapp, Telegram (not including name or any other info). Therefore this should not be a goal (and would not be feasible to implement).


# Proposed solution


## Key distribution


~~~ plantuml-utxt
participant Platform1 Client as A [fillcolor="orange"]
participant Platform1 Front End as B [fillcolor="orange"]
participant Platform1 Name Server as C [fillcolor="orange"]
participant Authoritative Platform2 Name Server as D [fillcolor="orange"]
participant Platform2 KDS as E [fillcolor="orange"]

C->D: Request Platform2 Name Records
D->C: Replicate Platform2 Name Records
A->B: PIR Query PN/UserID
B->C: PIR Query PN/UserID
C->B: Supported service IDs + default service
A-->E: Tunneled KDS query
B->E: Query PN/UserID
E->B: Return Public Key Bundle
E-->A: Tunneled query result
B->A: Return Public Key Bundle
A->A: Encrypt Message
~~~


Taking Platform1 client sending to a Platform2 user as an example:


1. Platform1 name server replicates authoritative Platform2 NS records. There will need to be a shared list of participating services and name server endpoints.

2. Platform1 client sends key bundle request to Platform1 front end (PIR encrypted PN/UserID)

3. Platform1 FE gets supported key distribution service IDs, version number + default service=Platform2 via PIR protocol from its own name server.

4. Platform1 FE queries Platform2 KDS to retrieve public keys. [^1]

*   4.1 Platform1 Client makes a tunneled KDS query to the Android KDS. Platform1 Client first sends (query and session key) encrypted w/ AM public key to Platform1 FE.
*   4.2 Platform1 FE sends encrypted query to Platform2 KDS
*   4.3 Platform2 KDS decrypts query and session key, encrypts response w/ session key
*   4.4 Platform2 KDS sends encrypted response to Platform1 FE
*   4.5 Platform1 FE forwards to Platform1 client

This provides E2EE interop while only disclosing to gateway service which services a phone number is registered to. In all other respects, gateway services learn no more information than in the non-interop case.


## Message receipt/delivery

A similar architecture can be used for message delivery

~~~ plantuml-utxt
participant Platform1 Client as A [fillcolor="orange"]
participant Platform1 Front End as B [fillcolor="orange"]
participant Platform1 Name Server as C [fillcolor="orange"]
participant Authoritative Platform2 Name Server as D [fillcolor="orange"]
participant Platform2 DS as E [fillcolor="orange"]
participant Platform2 Client as F [fillcolor="orange"]

C->D: Request Android Name Records
D->C: Replicate Android Name Records
A->B: E2EE Message payload + PIR encrypted PN/UserID
B->C: PIR Query PN/UserID
C->B: Supported service IDs + default service
A-->E: Tunneled Recipient Phone Number
A-->F: Tunneled Sender PN/UserID
B->E: E2EE Message payload
E->F: E2EE Message payload
~~~


1. Platform1 name server replicates authoritative Platform2 NS records
2. Platform1 client sends E2EE message payload + PIR encrypted PN/UserID to Platform1 front end.
3. Platform1 FE gets supported service IDs + version numbers + default service=Platform2 via PIR protocol. The version number ensures senders use compatible protocols in the case of changes.
4. Platform1 client tunnels recipient phone number to Platform2 Delivery Service (encrypted with Platform2’s pubkey)
5. Platform1 client tunnels sender phone number to AM client (using protocol similar to [Signal Sealed Sender](https://signal.org/blog/sealed-sender/))
6. Platform1 FE sends message to Platform2 DS after verifying that the default service signature matches.
7. Platform2 DS delivers message to AM client.


## Resolver registration

Each service is responsible for registering user enrollments with the resolver. 


## Preferred service integrity

While the preferred service is public, the user should control its value/integrity. Therefore to prevent anyone but the user modifying the default service value, records must be signed with the user’s private key and verified by the sender against their public key. For multiple key pairs across different services, the first key pair to sign the default service bit must be used to change the default.


~~~ plantuml-utxt
participant Client as A [fillcolor="orange"]
participant Matrix Bridge as B [fillcolor="orange"]
participant Android Bridge as C [fillcolor="orange"]
participant Whatsapp Bridge as D [fillcolor="orange"]
participant Resolver as E [fillcolor="orange"]

B->E: Register Preferred Service + Signature
A->E: Query PN/UserID
E->A: Return supported service IDs + default service preference + signature
A->A: Verify default service pref signature
~~~


## Cross-service identity spoofing

Today, a messaging service may support one or more ways of identifying a user including email address, phone number, or service specific user name.

Messaging interoperability introduces a new problem that traditionally has been resolvable at the service level: cross-service identity spoofing, where a user on a given E2EE may or may not be addressable at the same ID on another service due to a lack of global uniqueness constraints across providers.

As a result, a user may be registered at multiple services with the same handles, e.g. if Bob’s email is [bob@example.com](mailto:bob@example.com) and his phone number is 555-111-2222 and he’s registered with Signal and iMessage, he would be addressable at [bob@example.com](mailto:bob@example.com):iMessage, 555-111-2222:iMessage, and 555-111-2222:Signal. In this case, the same userId on iMessage and Signal is acceptable as the phone number can map to only one individual who proves their identity by validating ownership of the SIM card.

On services where a user can log in with a username _alone_, however e.g. Threema and FooService, the challenge becomes: 


*   Alice messages Bob at Bob's preferred service (bob@Threema)
*   Eve messages Alice impersonating Bob using bob@FooService
*   Alice needs some indicator or UI to know that bob@Threema isn't bob@FooSercice and that when bob@FooService messages, it should not be assumed that bob@FooService is bob@Threema.

One way to solve this would be storing the supported services for a contact in Contacts and if a receipt receives a message from an unknown sender, to treat it as spam or otherwise untrusted from the start.


# Privacy of resolver lookup queries

Resolver lookup queries leak the user’s social graph - i.e. who is communicating with whom, since the IP address of the querying client can be tied to user identity, especially when operating over a mobile data network. Therefore we propose to use Private Information Retrieval (PIR) to perform the resolver queries. We have evaluated multiple alternative schemes. The proposed solution is based on Google's implementation of PIR using Public Key PIR with sharded databases. Most recent public open source version is [here](https://github.com/google/private-retrieval). Cost estimates suggest this is feasible even for very large resolver databases (10 BN records).

## Proposed protocols

A private information retrieval protocol enables a client holding an index (or keyword) to retrieve the database record corresponding to that index from a remote server. PIR schemes have communication complexities sublinear in the database size and they provide access privacy for clients which precludes the server from being able to learn any information about either the query index or the record retrieved. A standard single-server PIR scheme provides clients with algorithms to generate a query and decode a response from the server. It also provides an algorithm for the server to encode a response.

Google [framework](https://www.usenix.org/conference/usenixsecurity23/presentation/patel) for Public Key PIR can be wrapped around any standard lattice-based PIR scheme. This framework consists of server database setup, client key initialization, client query generation, server response computation, and client response decryption sub-protocols.

   

### Server database setup



*   **Sharding**: If the database size is over 2<sup>20</sup>, shard it into sub-databases of ~1 million unique records each. We found ~1 million is a good balance for privacy and costs. Performing PIR over larger databases gives stronger privacy but is more costly. Similarly, running PIR queries over smaller databases is less costly but gives weaker privacy. 
    *   Sample a hash key **K<sub>s</sub>** for sharding.
    *   Using **K<sub>s</sub>**, shard the large database of **r** records into **⌈r/2<sup>20</sup>⌉** sub-databases based on the hash prefix of the record's unique identifier.
    *   **N.B.** The hash key will be provided to clients to determine the sub-database to query. 
*   **Set partitioning boundaries for each sub-database D**: Given a **n** key-value pairs sub-database **D = {(k<sub>1</sub>,v<sub>1</sub>),...,(k<sub>n</sub>,v<sub>n</sub>)}**, then
    *   Compute the number of database partitions as **b = n/d<sub>1</sub>**. Where **d<sub>1</sub>** is the desired size for each partition. A value of 128 for **d<sub>1</sub>** works well.
    *   Sample a partitioning boundary hash key **K<sub>1 </sub>** to generate a target partition for each sub-database key.
    *   Compute the hash **F<sub>1</sub>(K<sub>1</sub>,k<sub>i</sub>)** for each record identifier **k<sub>i</sub>** 
    *   Sort the hashes and then divide them into **b** partitions **P<sub>1</sub>,...,P<sub>b</sub>**.
    *   Store the b partition boundaries beginning hash values **B<sub>0</sub>, …, B<sub>b</sub>**. Note that **B<sub>0</sub> = 0**, and **B<sub>b</sub> = |U| -1** where U is the rage for **F<sub>1</sub>(K<sub>1</sub>,k<sub>i</sub>).**
    *   **N.B.** The partition boundaries will be provided to clients and can be stored efficiently (e.g., ~11KB for **n** = 2<sup><strong>20</strong></sup>, **d<sub>1</sub>** = 128, **|U|** = 2<sup><strong>64</strong></sup>).
*   **Encode each sub-database**: Sample two hash keys **K<sub>2</sub>** and **K<sub>r</sub>** where **K<sub>2</sub>** will be used to generate a row vector within each partition, and **K<sub>r</sub>** is used to generate a representation for the encoded database as **F(K<sub>r</sub>,k<sub>i</sub>)||v**.
*   **N.B.** **F(K,k)** is the range value from hashing **k** with key **K** and **||** is a concatenation.
*   For each partition **P<sub>i</sub>**
    *   Construct a **|P<sub>i</sub>| x d<sub>1</sub>** Platform1 **M<sub>i</sub>** by appending a random row vector from the bit vector derived from **(F<sub>2</sub>(K<sub>2</sub>,k||1),...,F<sub>2</sub>(K<sub>2</sub>,k||d<sub>1</sub>))**.
    *   Construct a **|P<sub>i</sub>|** vector **y<sub>i</sub>** by appending **F<sub>r</sub>(K<sub>r</sub>,k)||v** for each **(k,v)** in **P<sub>i</sub>**
    *   Solve for **e<sub>i</sub>** that satisfies **M<sub>i</sub>e<sub>i</sub> = y<sub>i</sub>**.
*   Construct the encoded **d<sub>1</sub> x b** Platform1 as **E = [e<sub>1</sub> … e<sub>b</sub>]**
*   The Platform1 **E** is the encoded Platform1 for sub-database **D**. 
*   The clients must download parameters **(K<sub>1</sub>,K<sub>2</sub>,K<sub>r</sub>)** to query each sub-database, plus **K<sub>s</sub> **to inform the server of the target sub-database for a query**.**

This offline protocol is completed by the server before answering any client query. Note that a sub-database must be re-encoded after an update. Encoding only takes a few minutes.

### Client key initialization



*   The client generates a per-key unique identifier (**UID**),  **private key** and **public key** using a fully homomorphic encryption (FHE) scheme relying on some hardness assumptions, such as a Ring Learning with Errors problem.
*   The client persists the **UID** and **private key** into its local key store, and uploads query-independent parameters **UID** and **public key** to the server. These later parameters will enable the server to perform cryptographic operations (i.e., FHE) efficiently. 
*   The server needs to maintain an up-to-date mapping of **UID** to **public key** for all clients. 
*   Each client completes this offline initialization protocol before running any query. It also needs to perform it periodically (e.g., weekly or monthly) to prevent server linkability of private queries to the user over an extended period.
*   The client download query parameters from the server:
    *   Sharding hash key **K<sub>s</sub>** to inform the server of the target sub-database for a query.
*   Sets of parameters (**K<sub>1</sub>,K<sub>2</sub>,K<sub>r</sub>,B<sub>0</sub>, …, B<sub>b</sub>**) for each sub-database. 

### Client query generation



*   The client creates a query to retrieve the value corresponding to a database key **k** as follows:
*   Compute **d = F<sub>s</sub>(K<sub>s</sub>,k)** to identify the sub-database to query.
*   Compute **j = F<sub>1</sub>(K<sub>1</sub>,k)** to learn which partition contains the desired entry from the downloaded partition boundaries for the sub-database.
*   Generate **z** vector **v** of length **d<sub>1</sub> , … , d<sub>z</sub>** . Compute a **d<sub>1</sub>**-length random bit vector **v<sub>1</sub>** from **(F<sub>2</sub>(K<sub>2</sub>,k||1),...,F<sub>2</sub>(K<sub>2</sub>,k||d<sub>1</sub>))**. Compute **v<sub>2</sub>** as a zero bit vector of **d<sub>2</sub>** length with only the bit set at **⌊j/⌈n/d<sub>1</sub>d<sub>2</sub>⌋**. Similarly compute **v<sub>3</sub> , … , v<sub>z</sub>**.
*   Finally use the underlying PIR scheme and the private key to encrypt the **z** vector **v.**
*   Send **v, d** and the **UID** to the server. 
*   **N.B.** The dimension **d<sub>z</sub>** is typically small; a size of 2 or 4 works well.

### Server response computation



*   The server retrieves the public key for the client's **UID**, and computes the ciphertext of the value corresponding to the key of interest for the sub-database identifier **d** as follows.
*   Take the encoded sub-database **E** as a **d<sub>1 </sub>x ⌈n/d<sub>1</sub>⌉** Platform1 **E<sub>1</sub>**, use the underlying PIR response answering algorithm to compute **v<sub>1</sub>.E<sub>1</sub>**, and rearrange the resulting **⌈n/d<sub>1</sub>⌉** vector as a **d<sub>2 </sub>x ⌈n/d<sub>1</sub>d<sub>2</sub>⌉** Platform1 **E<sub>2</sub>**. 
*   Next, compute **v<sub>2</sub>.E<sub>2</sub>**, and rearrange the resulting **⌈n/d<sub>1</sub>d<sub>2</sub>⌉** vector as a **d<sub>3 </sub>x ⌈n/d<sub>1</sub>d<sub>2</sub>d<sub>3</sub>⌉** Platform1 **E<sub>3</sub>**. 
*   The server similarly repeats the computation for the remaining dimensions **v<sub>3</sub> ,… , v<sub>z</sub>**.
*   The end result is a ciphertext **r** of the database value corresponding to **k**. The server sends **r** back to the client.

 

### Client response decryption



*   The client uses the underlying PIR response decoding algorithm and **private key** to decrypt the response **r** as **k<sub>r</sub>||v**. If **F<sub>r</sub>(K<sub>r</sub>,k) == k<sub>r</sub>** then return **v** otherwise return **⟂** (key not found). 

## FHE key requirements


*   FHE with key offering at least 128-bit of security
*   ElGamal, NIST P-224r1 curve and a 4 bytes plaintext size for fast decryption.
*   Gentry–Ramzan, used a 2048-bit modulus
*   Damgård–Jurik, used 1160-bit primes

## Cost estimates

In these estimates, we propose using sub-databases of size ~1 million of identifiers. We believe this is a good balance between the privacy and usability of the PIR schemes. For 1.28 TB (10 billion records), we break this down into 10,000 sub-databases each of size 1 million records this gives a cost estimate for each query as below:


| Parameter                                                | Cost estimate     |
|----------------------------------------------------------|-------------------|
| PIR Public Key Size Per Device (key storage required)    | 14 MB             |
| Upload Bandwidth Per Query                               | 14 KB             |
| Download Bandwidth Per Query                             | 21 KB             |
| Client Time Per Query                                    | 0.1s              |
| Server Time Per Query (Single Thread)                    | 0.8-1s            |



Note on some assumptions for feasibility:



1. Resolver queries will be cached (vs requiring a roundtrip for every message) and asynchronous with message sending, therefore 1s latency is acceptable
2. It is acceptable for key changes to be communicated reactively on decrypt failure.
3. Group messaging E2EE is bootstrapped using individual users’ public keys and for V1, group state will be stored by the initiating user’s service provider. Therefore no additional discovery mechanism is required.

## Notes

[^1]:

     Clients may cache service Ids + default service + keys for a given User ID; discussion and best practices of such caching will be discussed in a future document.


# Conventions and Definitions

{::boilerplate bcp14-tagged}



# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

The technical description of the private information retrieval is based on Sarvar Patel, Joon Young Seo and Kevin Yeo's USENIX Security '23 paper titled ["Don’t be Dense: Efficient Keyword PIR for Sparse Databases
"](https://www.usenix.org/conference/usenixsecurity23/presentation/patel).
