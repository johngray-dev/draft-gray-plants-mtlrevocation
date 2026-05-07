---
title: "MTL Revocation"
abbrev: "MTL Revocation"
category: std

docname: draft-gray-plants-mtlrevocation-latest
ipr: trust200902
area: Security
wg: LAMPS
kw: Internet-Draft

keyword:
 - MTL
 - Revocation
 - Merkle
 - Ladder
 - OCSP
 - CRL

venue:
group: WG
type: Working Group
mail: WG@example.com
arch: https://example.com/WG
github: USER/REPO
latest: https://example.com/LATEST

author:
-
    ins: J. Gray
    name: John Gray
    org: Entrust Limited
    abbrev: Entrust
    street: 2500 Solandt Road – Suite 100
    city: Ottawa, Ontario
    country: Canada
    code: K2K 3G5
    email: john.gray@entrust.com
-
    ins: J. Harvey
    name: Joseph Harvey
    org: Verisign Labs
    abbrev: Verisign
    street: 12061 Bluemont Way
    city: Reston, VA
    country: United States of America
    code: 20190
    email: jsharvey@verisign.com
 

normative:
RFC2119:
RFC5280:
RFC6960:
RFC8174:

informative:
I-D.draft-harvey-cfrg-mtl-mode-02



--- abstract

This document defines Merkle Tree Ladder Revocation which makes use of a Merkle Tree Ladder structure to compute and distribute revocation information more efficiently than existing revocation structures such as Certificate Revocation Lists (CRL) and Online Certificate Status Protocol (OCSP).

--- middle

# Introduction

When X.509 Certificate validation is performed, a revocation check operation is needed to ensure that the certificate being validated has not been revoked by its issuing CA.  Where CRL’s are used as the method of revocation, they can grow to be very large (10s of Megabytes), which is inefficient from a bandwidth perspective.   Partitioned CRL’s are better in that they reduce bandwidth, but require multiple signatures.   OCSP is another type of revocation that replaces a list of revocations with an individual OCSP response.  The issue with OCSP is that a signature is required for every response.    OCSP has the ability to enable real-time responses by use of a nonce, but in practice having a freshly signed response for every revocation request becomes too onerous for OCSP responders for very large CA systems.

This proposal takes a different look at how revocation can be performed using the Merkle Tree Ladder (MTL) approach.   For revocation use-cases each leaf node in the Merkle Tree Ladder represents a signature over a certificate entry.   Efficiency is enabled because only a single signature over the set of ladders is required to validate any response in the MTL structure which can represent all revocation information for a CA.  The computation of each signature is effectivly replaced by a hash, and the MTL root signature can be cached offering minimal download overhead, especially in the context of larger cryptographic signatures schemes.  This can allow orders of magnatude faster compution of pre-computed revocation proofs which can lead to more timely revocation information.

Dividing up work in different ways for optimal environment usage.  Eg.  Sigs turn to hashes, HSM may not need to sign as much.

## Terminology {#sec-terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}}  {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

The following terms are used in this document:

PKI:
Public Key Infrastructure, as defined in \[RFC5280].

SIGNATURE:
A digital cryptographic signature, making no assumptions
about which algorithm.

CRL:
A Certificate Revocation List, as defined in \[RFC5280]

OCSP:
Online Certificate status protocol, as defined in \[RFC6960]



# Revocation Request Structure {#sec-request-structure}

To request revocation information, at minimum information identifying the certificate whose status is being checked is required.  In a PKI system, a certificate can be uniquely identified by the issuer of the certificate along with its unique serial number.  According to \[RFC5280] the serial number MUST be unique for each certificate issued by a given CA   This can be seen in OCSP \[RFC6960] where the information used to identify a certificate in a request is a CertID:

~~~
Request ::=     SEQUENCE {
  reqCert                    CertID,
  singleRequestExtensions    \[0] EXPLICIT Extensions OPTIONAL }

CertID ::= SEQUENCE {
   hashAlgorithm      AlgorithmIdentifier,
   issuerNameHash     OCTET STRING, -- Hash of Issuer's DN
   issuerKeyHash      OCTET STRING, -- Hash of Issuers public key
   serialNumber       CertificateSerialNumber }
~~~

For CRL's, the revocation information for a Certificate is usually contained in a Certificate Revocation list Distribution point extension (CRLDP) which identifies the location of the CRL.  The CRL can contain revocation information for all certificates in the PKI, or a subset of certificates in the PKI.

## Merkle Tree Ladder Revocation Structure

A Merkle Tree Ladder optimized for use for Revocation gives the best properties from both the OCSP approach and CRL approach.

* Each leaf node in a MTL structure can represent the hash over the certificate status information.

### Merkle Tree Ladder Generation Procedure {#sec-generation}

1. For each certificate:
   Compute the Status Information (TBD some DER encoded structure, could be the OCSP response structure for example)
   Hash the DER encoded structure
   Add hash to the growing MTL structure
2. Generate the signature on the Ladder - Section section on suggested algorithms
3. For each certificate response:
   a. Generate an MTL Condensed Signature (authentication path and handle reference value)
   b. Include the condensed signature in the certificate response
   c. Map the certificate identifier information to the included response information

Note:  Implementations will need to associate the Certificate Identifier information with the certificate response object so that when request information is received, the response information for that associated responsed can be located.

### Validation of a Response

* A certificate extension (for example AIA id-ad-ocsp, or new defined AIA extension) is used to convey how a client using the certificate's revocation information can check the MTL signature

  * TODO probably a new extension will be needed, the client would need to retreive at least the MTL condensed signature, and possibly the ladder signature if it didn't already have it.   Clients can cache the ladder signature for future use, or could periodically poll to download the latest ladder signature.
* Using a Condensed signature would reduce size increase of comparable OCSP responses.   Even with the occasional update of the MTL ladder signature, sizes would we much smaller than combined CRL's.
* Client will reconstitute the full MTL signature (authentication path, handle reference value and signature) and verify it.

# ASN.1 Structures

TODO - This section will describe the required ASN.1 structures

# MTL Common Use Cases

TODO



## Adding new Certificate status information to an existing Ladder

After the initial computation of the MTL structure that represents all the revocation information for all the certificates in a PKI system, additional revocation responses will need to be added as new certificates are generated or revoked in the PKI.

When a new certificate is created, the hash of the certificate status information can be added to the Merkle Tree Ladder as specified in {{sec-generation}} for that particular certificate.  However, it should be noted that the signature over the root node that covers all revocation information will need to be recomputed.  This procedure will not invalidate any exisitng revocation response information, but it will require that the new root signature be downloaded and cached for any client that requests revocation information for this newly added or updated certificate status.  If this process is continued on a certificate by certificate basis, then the efficiency of using this MTL structure is reduces.  Therefore, the following can be used to mitigate the decrease in efficiency over time:

1. Certificate status information can be published in batched time periods to increase the efficiency.  This would have the effect of revocation information not becoming immediatly available at the cost of efficiency.   For example, if the batch period was 10 minutes, and the PKI issued 5 new certifiates and 1 revocation in that time period, the MTL structure could be updated at 10 minutes to include all 6 new certificate status information.  If no new certificate status information is created in the batch period, the MTL structure would not need to be updated.
2. PKI systems may have times where many certificates are issued or revoked at once.  It would be possible to set both a batching time period as well as a status information upper bound to trigger an update to the Merkle tree.  For example, if the batch period was 10 minutes, and the status information update was set to 100, and the PKI issued 97 new certifcates and revoked 3 in a span of 7 minutes, the Merkle Tree ladder could be triggered to add the 100 new revocations immediately and restart the timer.  This would have the effect of giving more timely response information when a PKI has heavier load, and would spread out the computation required over time.
3. It may be desirable to periodically re-calculate the entire MTL structure to clean up status information that has been updated and is no longer valid.
4. The MTL structure will need to take into account the validity period of each response.



## Updating status information for an existing certificate

When the revocation status information for an existing certificate already contained in the MTL structure changes, the MTL structure simply adds a new entry as defined in {{sec-generation}}.  This has the effect of invalidating the existing status for that certificate which is already contained in the MTL structure.  The information at the MTL Revocation server's side which maps the certificate identifier information to that revocation in the MTL structure MUST be updated so that a client requesting certificate status information gets access to the most relevent revocation information.

## Full MTL Signature versus partial MTL signature (Generating a Complete MTL signature from a Partial signature)

A full MTL signature consists of the authentication path from the message to the ladder as well as the ladder, and the underlying signature on the ladder.

A partial MTL signature consists of only the authentication path from the message to the ladder as well as the ladder.

When a client requests certificate status information for a certificate, it needs the following pieces of information

1. The Certificate identifier (CertID) of the certificate it is requesting as specified in {{sec-request-structure}}
2. A HASH of its most recently cached MTL signature on the ladder or Null if it does not have one.
TODO:  Does this requirement make the client too stateful?   If the client always has to request the full path, then no efficiency is gained at the client side.

The MTL Revocation server can use this information to determine whether a full MTL signature is required to be sent, or if the client has enough information to construct a full MTL signature from the most recent cached MTL signature.

EDNOTE:  Does regular cache of CRL's OCSP give just as much an efficiency gain?   It would be expected that a client would use regular caching (if they have already received a responce for the specified certificate and its validity period is still valid).  If the validity period has expired, then a new certificate status is needed, and the updated full chain is needed.   Efficiency is only gained at the client when it needs to check revocation of many certificates from the same PKI infrastructure.



# MTL and Real Time Responses

TODO


# Conventions and Definitions

{::boilerplate bcp14-tagged}



# Security Considerations

TODO Security



# IANA Considerations

This document has no IANA actions.


