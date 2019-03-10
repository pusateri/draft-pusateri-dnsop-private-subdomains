---
title: Private DNS Subdomains
docname: draft-pusateri-dnsop-private-subdomains-00
date: 2019
ipr: trust200902
area: Internet
wg: DNSOP Working Group
kw: Internet-Draft
cat: exp
stand_alone: false

coding: utf-8
pi:
  - toc
  - symrefs
  - sortrefs

author:
  -
    ins: T. Pusateri
    name: Tom Pusateri
    org: Unaffiliated
    street: ""
    city: Raleigh
    code: NC 27608
    country: USA
    phone: +1 (919) 867-1330
    email: pusateri@bangj.com

--- abstract

This document describes a method of providing private DNS subdomains such that each subdomain can be shared among multiple devices of a single owner or group. A private subdomain can be used for sharing personal services while increasing privacy and limiting knowledge of scarce resources.

--- middle

# Introduction

Section 6.6 of {{?RFC7558}} highlights the privacy risks of DNS service announcements in clear text. While there has been a long focus on access control to private services including the use of encryption and authentication through TLS {{!RFC8446}} for connections, the DNS-SD announcements {{?RFC6763}} themselves may leak private information including but not limited to device types and versions, enabled services on the device subject to attack, personal names and identifiers used for tracking, etc.

Some services are meant to be advertised and used freely by devices on the link but other services are restricted to an owner or group and these services are announced publicly as a side effect of the current service discovery deployment.

This document defines a method for collaborating devices to share private services with one another but without revealing the existence of these services in a public way. This provides an additional layer of privacy protection for an individual's devices or those of a defined group sharing a common purpose.

The additional privacy is achieved by creating private subdomains that require a private key for bidirectional access to DNS queries and responses for the zone.

This document defines a subdomain hierarchy for providers to enable this feature as well as a mechanism for interoperable transfers to and from the subdomain. This includes creating and destroying the subdomains, adding and removing records through DNS Update, and techniques for private authenticated queries and responses.

# Requirements Language

{::boilerplate bcp14+}

# Subdomain Operations

An administrative domain provider will enable private subdomains by creating a base subdomain at `_pvt.<domain>.` In addition to the SOA and NS records, a public KEY record {{!RFC2535}}, {{!RFC3445}} MUST be created at the apex which is openly available for queries. This public key will be used for message encryption relating to the maintenance of private subdomains.

The Zone bit for all KEY records used in this specification MUST be set to 0. The protocol value for the KEY record is set to DNSSEC (3) and the algorithm value is taken from the available algorithms defined in the IANA DNS Security Algorithm Numbers. Only algorithms that can be used for encryption are eligible for use in the KEY record for this purpose.

## Domain name encryption {#dname}

Domain names in questions, responses, and updates MUST provide the subdomain name in clear text to correctly route the query and cache the responses in resolvers along the path. However, labels before the subdomain name are encrypted to provide privacy. Therefore, domain names must be encoded and decoded at the sender and receiver of the DNS message.

Only the owner and the administrative domain provider know the true contents of the records. Intermediary resolvers or forwarders simply see the encrypted questions or answers as normal cacheable questions or response records and can effectively return cache results like any other records without being able to understand the true contents.

The QTYPE of a question and the RRTYPE in a response record are also encoded in the name to further increase privacy and reduce record type optimizations that might assume particular formats of RDATA for known record types. For example, if an `A` record type was used, some implementations MAY assume 4-byte RDATA and get confused by an encrypted version that contains more bytes. Therefore, a Encrypted Resource Record type (ENCR) is defined in {{ENCR}} which is returned in all answers. The original QTYPE and RRTYPE are prepended to the original QNAME in the question or owner name in the response before encrypting. The QTYPE is then always set to ENCR and the answers are naturally returned with this type as well.

The subdomain suffix is first removed and the remaining portion of the original domain name is prepended with the QTYPE or RRTYPE and encrypted with the public key found in the apex of the `_pvt.<domain>.` Next the subdomain labels are appended in clear text to the encrypted portion of the name and type.

Since there is a limit of 63 octets to a label, it is possible that the encrypted portion of the name could surpass this limit. Further work and experimentation MAY be needed to arrive at the best solution to this issue and this document will be updated at that time. The current solution is to include an initial byte after the length of the first label containing the number of labels that make up the encrypted portion of the name. Those labels will be concatenated (excluding the length bytes) to form the encrypted portion of the name. This encoding and decoding is only performed by the querier and the authoritative server responding to the query and is transparent to intermediate forwarders and resolvers.

## Zone Creation

A subdomain is created by the owner in an administrative domain for which the owner has a trusted relationship. For instance, if a user has services provided by an administrative domain and that user has been assigned a user name or account at that domain, the user could then uniquely claim ownership of the subdomain `<user>._pvt.<domain>.`

The administrative domain can use any means it finds sufficient to verify the trusted relationship with the user including RADIUS, AAA, email verification, or some other method not described here which may include enabling the service on a per-user basis.

The user can then create the subdomain by sending an UPDATE {{!RFC2136}} to the MNAME of the SOA record for `_pvt.<domain>.` containing a new KEY record for the zone `<user>._pvt.<domain>.` The KEY record contains a public key that will later be used by the administrative domain to verify signed requests and send encrypted responses. If this zone already exists, the UPDATE fails with RCODE YXRRSet. If the zone does not exist, the user has successfully claimed the zone and all subsequent operations by the user will require knowledge of the private key associated with the registered public key in the KEY record. The user can distribute this private key to any or all of its devices for access to the private subdomain.

In response, appropriate NS records for `<user>` will be created in the `_pvtr.<domain>.` and a new SOA record `<user>._pvt.<domain>.` with appropriate record fields will also be created along side the new KEY record submitted by the user. The authoritative servers for the new `<user>._pvt.<domain>.` will be managed by the administrative domain provider.

At this point, the owner has knowledge of the public key of `_pvt.<domain>.` and the administrative domain has knowledge of the public key of `<user>._pvt.<domain>.` All subsequent operations for the subdomain `<user>._pvt.<domain>.` use encrypted records based on the public key of the message receiver. All operations by the owner MUST included a signature using the private key which can be verified by the receiver using the owners public key. In this way, the records are only ever updated or revealed to an authorized user.

## Adding / Removing Resource Records

Once the zone is created, the user can begin adding or removing records to the `<user>._pvt.<domain>` subdomain through additional UPDATE messages for the zone. The subdomain in the Zone section of the UPDATE appears as usual in plain text. However, individual records in the Update section of the message are encrypted with the administrative domain's public KEY found at `_pvt.<domain>.`
In order to encrypt the records to be added, they will be encoded in a new resource record type defined below called an Encrypted Resource Record {{ENCR}}. These records use the same CLASS as the encapsulated record and contain an algorithm type to indicate how they can be decrypted.

To ensure zone additions and deletions are from the subdomain owner, the UPDATE message must be signed with the owner's private key and the signature included in the additional data section in the form of a SIG(0) record {{!RFC2931}}. The owner MUST sign the encrypted form of the records in the Update section in order for the receiver to validate the UPDATE message before attempting to decrypt the records contained within. More information about authenticating UPDATE messages can be found in {{!RFC3007}}.

The administrative domain provider can then use the private key associated with the public key stored in the KEY record at `_pvt.<domain>.` to individually decrypt the resource records contained in the Update section. If the records can be successfully decrypted, they are added to, or removed from, the `<user>._pvt.<domain>.` zone. If the records cannot be decrypted, the receiver MUST return FORMERR. See section 2.5.2 of {{!RFC2136}} for the specifics of adding or removing records in the Update section.

In order to change the KEY record at the apex of the zone, the old KEY record should be added to the Prerequisite section and the new KEY record in the Update section. The entire UPDATE message MUST be signed by placing the SIG(0) signature record in the Additional data section.

This is useful for changing the algorithm or key length for the public/private key pair.

If instead the private key associated with the public key in the KEY record has been compromised, the user should contact the administrative domain out of band to verify authenticity and replace the KEY record through a process established by the provider.

## Zone Destruction

When the owner is ready to destroy the subdomain `<user>._pvt.<domain>.`, it can do so by deleting the KEY record at the apex through an UPDATE message. As above, the UPDATE message additional data section MUST contain a SIG(0) signature over the entire message. Once validated, the zone will be removed along with the NS records for `<user>` in the `_pvt.<domain>` zone. The KEY record containing the public key is not encrypted.

The owner is free to destroy and re-create the subdomain as needed for as long as the relationship continues with the administrative domain.

# Querying Resource Records

The question section of a DNS query contains the QNAME, QTYPE, and QCLASS. The original QNAME and QTYPE are encoded into a new encrypted form and used as the QNAME of the outgoing query. The outgoing QTYPE is always `ENCR`. The outgoing QCLASS is not encoded and contains the class of the original query.

## Signed Requests

All queries MUST be signed with the private key of the owner. The signature MUST be in the Additional records section in the form of a SIG(0) record. If a query is received that is not signed or cannot be verified with the public key in the KEY record at the apex of the `<user>._pvt.<domain>.`, a response containing the question as received with an RCODE of REFUSED with no answers MUST be returned. The Authority section of the response MUST contain the SOA record for the `<user>._pvt.<domain>.` in clear text.

Authoritative servers MUST verify the signature in the query before decoding the QNAME and returning the results. It is not necessary for intermediate caching resolvers to verify the signature in the query since any cached answers are already encrypted.

## Encrypted Questions

When a query is received by the authoritative server for the zone, the private key corresponding to the public key stored in the KEY record at `_pvt.<domain>` is used to decrypt the question. Only the authoritative servers have the ability to decrypt the query. The original question is recreated and matching records are found.

# Responses

Responses include the encrypted version of the question (with the subdomain in clear text), along with any matching answers represented as Encrypted Resource Records {{ENCR}}. Likewise, the owner name for each record in the answer section includes the encrypted label(s) and the subdomain part in clear text.

## Signatures in Responses

Responses MUST NOT be signed because the receiver is not able to determine which public key to use to verify the signature. The response could come from the authoritative server or it could come from an intermediate caching resolver.

Future work will attempt to provide a way to ensure response integrity.

## Encrypted Records

Each resource record included in the response is encrypted by the authoritative server using the public key in the KEY record at the apex of `<user>._pvt.<domain>.` These responses can be cached by resolvers and returned  in subsequent queries using normal DNS operations.

The encrypted owner name of an `ENCR` record is decrypted to obtain the actual owner name and RRTYPE. The `ENCR` RDATA is decrypted to obtain the actual RDATA of the original record. The format of the `ENCR` RDATA provides algorithm information sufficient for decryption.

# Encrypted Resource Record (ENCR) {#ENCR}

The format of an ENCR record is defined here for use in the IANA registry of DNS Resource Record Types. It contains the encrypted output of an existing resource record along with the algorithm used for encryption. The combination of the contents of this resource record along with the public key in the KEY record is sufficient to decode the encrypted contents to obtain the origin resource record.

List encryption algorithms here.

## Encrypted Resource Record RDATA format

Show RDATA packet format here.

# Security Considerations

Since responses are not signed, there is a opportunity for the message contents to be tampered with. Fake records can not be created but records could be removed from the Answer section and header fields could be modified.

----

--- back

