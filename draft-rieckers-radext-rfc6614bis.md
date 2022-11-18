---
title: "Transport Layer Security (TLS) Encryption for RADIUS"
abbrev: "RADIUS over TLS"
category: std

obsoletes: 6614

docname: draft-rieckers-radext-rfc6614bis-latest
submissiontype: IETF
v: 3
area: "Operations and Management"
workgroup: "RADIUS EXTensions"
keyword:
  - RADIUS
  - TLS
venue:
  group: "RADIUS EXTensions"
  type: "Working Group"
  mail: "radext@ietf.org"

author:
  - name: Jan-Frederik Rieckers
    org: Deutsches Forschungsnetz | German National Research and Education Network
    street: Alexanderplatz 1
    code: 10178
    city: Berlin
    country: Germany
    email: rieckers@dfn.de
    abbrev: DFN
    uri: www.dfn.de
    role: editor
  - name: Stefan Winter
    org: Fondation Restena | Restena Foundation
    street: 2, avenue de l'Université
    code: 4365
    city: Esch-sur-Alzette
    country: Luxembourg
    email: stefan.winter@restena.lu
    abbrev: RESTENA
    uri: www.restena.lu


normative:

informative:



--- abstract

This document specifies a transport profile for RADIUS using Transport Layer Security (TLS) over TCP as the transport protocol.
This enables dynamic trust relationships between RADIUS servers as well as encrypting RADIUS traffic between servers using a shared secret.

--- middle

# Introduction

The RADIUS protocol {{!RFC2865}} is a widely deployed authentication and authorization protocol.
The supplementary RADIUS Accounting specification {{!RFC2866}} provides accounting mechanisms, thus delivering a full Authentication, Authorization, and Accounting (AAA) solution.
However, RADIUS has shown several shortcomings, especially the lack of security for large parts of its packet payload.
RADIUS security is based on the MD5 algorithm, which has been proven to be insecure.

The main focus of RADIUS over TLS is to provide a means to secure the communication between RADIUS/TCP peers using TLS.
The most important use of this specification lies in roaming environments where RADIUS packets need to be transferred through different administrative domains and untrusted, potentially hostile network.

There are multiple known attacks on the MD5 algorithm that is used in RADIUS to provide integrity protection and a limited confidentiality protection. RADIUS over TLS wraps the entire RADIUS packet payload into a TLS stream and thus mitigates the risk of attacks on MD5.

Because of the static trust establishment between RADIUS peers (IP address and shared secret), the only scalable way of creating a massive deployment of RADIUS servers under the control of different administrative entities is to introduce some form of a proxy chain to route the access requests to their home server.
This creates a lot of overhead in terms of possible points of failure, longer transmission times, as well as middleboxes through which authentication traffic flows.
These middleboxes may learn privacy-relevant data while forwarding requests.
The new features in RADIUS over TLS add a new way to identify other peers, e.g., by checking a certificate for the issuer or other certificate properties, but also provides a simple upgrade path for existing RADIUS connection by simply using the shared secret to authenticate the TLS session.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

Within this document we will use the following terms:

RADIUS/TLS node:
: a RADIUS-over-TLS client or server

RADIUS/TLS Client:
: a RADIUS-over-TLS instance that initiates a new connection

RADIUS/TLS Server:
: a RADIUS-over-TLS instance that listens on a RADIUS-over-TLS port and accepts new connections

RADIUS/UDP:
: a classic RADIUS transport over UDP as defined in {{RFC2865}}

## Changes from RFC6614

Currently, there are no big changes, since this is just a restructured spec from {{?RFC6614}}.

The following things have changed:

Required TLS versions:
: TLS 1.2 is now the minimum TLS version, TLS 1.3 is included as recommended.

TLS compression:
: {{RFC6614}} allowed usage of TLS compression, this document forbids it.

TLS-PSK support:
: {{RFC6614}} lists support for TLS-PSK as OPTIONAL, this document changes this to RECOMMENDED.

Mandatory-to-implement(MTI) cipher suites:
: Following the recommendation from {{!RFC7525}}, the RC4 cipher suite is no longer included as SHOULD, and the AES cipher suite is the new MTI cipher suite, since it is the MTI cipher suite from TLS 1.2.
  Additionally, this document references {{RFC7525}} for further recommendations for cipher suites.

The following things will change in future versions of this draft:

* Usage of Server Name Indication
* More text for TLS-PSK

# Transport layer security for RADIUS/TCP

This section specifies the way TLS is used to secure the traffic and the changes in the handling of RADIUS packets.

## TCP port and Packet Types

The default destination port number for RADIUS over TLS is TCP/2083.
There are no separate ports for authentication, accounting, and dynamic authorization changes.
The source port is arbitrary.

## TLS Connection setup


{:jf: source="Janfred"}

The RADIUS/TLS nodes first try to establish a TCP connection as per {{!RFC6613}}.
Failure to connect leads to continuous retries.
It is RECOMMENDED to use exponentially growing intervals between every try.

After completing the TCP handshake, the RADIUS/TLS nodes immediately negotiate a TLS session.
The following restrictions apply:

* Support for TLS 1.2 {{!RFC5246}} is REQUIRED, support for TLS 1.3 {{!RFC8446}} is RECOMMENDED.
  RADIUS/TLS nodes MUST NOT negotiate TLS versions prior to TLS 1.2
* Support for certificate-based mutual authentication is REQUIRED.[^1]{:jf}
* Negotiation of mutual authentication is REQUIRED.[^2]{:jf}
* The RADIUS/TLS nodes MUST NOT offer or negotiate cipher suites which do not provide confidentiality and integrity protection.
* The RADIUS/TLS nodes MUST NOT negotiate compression.
* When using TLS 1.3, RADIUS/TLS nodes MUST NOT use early data ({{RFC8446}}, Section 2.3)
* RADIUS/TLS nodes SHOULD support TLS-PSK mutual authentication {{!RFC4279}}
* RADIUS/TLS implementations MUST, at minimum, support negotiation of the TLS_RSA_WITH_AES_128_CBC_SHA cipher suite and SHOULD follow the recommendations for supported cipher suites in {{RFC7525}}, Section 4.
* In addition, RADIUS/TLS implementations MUST support negotiation of the mandatory-to-implement cipher suites required by the versions of TLS they support.

[^1]: To me, it is not exactly clear if RADIUS/TLS implementations only need to support it or if the nodes actually need to do it (e.g. if it is allowed to configure a client to accept anonymous clients)
[^2]: Same comment as before.

Details for peer authentication are described in {{TLSPeerAuth}}.

After successful negotiation of a TLS session, the RADIUS/TLS peers can start exchanging RADIUS datagrams.
The shared secret to compute the (obsolete) MD5 integrity checks and attribute obfuscation MUST be "radsec".

## TLS Peer Authentication {#TLSPeerAuth}

The authentication of peers can be done using different models, that will be described here.

### Authentication using X.509 certificates with PKIX trust model

All RADIUS/TLS implementations MUST implement this model, following the following rules:

* Implementations MUST allow the configuration of a list of trusted Certificate Authorities for incoming connections.
* Certificate validation MUST include the verification rules as per {{!RFC5280}}.
* Implementations SHOULD indicate their trusted Certification Authorities (CAs). See {{RFC5246}}, Section 7.4.4 and {{!RFC6066}}, Section 6 for TLS 1.2 and {{RFC8446}}, Section 4.2.4 for TLS 1.3.
* Peer validation always includes a check on whether the locally configured expected expected DNS name or IP address of the server that is contacted matches its presented certificate. [^3]{:jf}
  DNS names and IP addresses can be contained in the Common Name (CN) or subjectAltName entries.
  For verification, only one of these entries is to be considered.
  The following precedence applies:
  for DNS name validation, subjectAltName:DNS has precedence over CN; for IP address validation, subjectAltName:iPAddr has precedence over CN.
  Implementors of this specification are advised to read {{?RFC6125}}, Section 6, for more details on DNS name validation. [^4]{:jf}
* Implementations MAY allow the configuration of a set of additional properties of the certificate to check for a peer's authorization to communicate (e.g., a set of allowed values in subjectAltName:URI or a set of allowed X.509v3 Certificate Policies).
* When the configured trust base changes (e.g., removal of a CA from the list of trusted CAs; issuance of a new CRL for a given CA), implementations MAY renegotiate the TLS session to reassess the connecting peer's continued authorization.[^5]{:jf}

### Authentication using certificate fingerprints

RADIUS/TLS implementations SHOULD allow the configuration of a list of trusted certificates, identified via fingerprint of the DER encoded certificate octets.
When implementing this model, support for SHA-1 as hash algorithm for the fingerprint is REQUIRED, and support for the more contemporary has function SHA-256 is RECOMMENDED.

### Authentication using TLS-PSK

The support for TLS-PSK is OPTIONAL.[^6]{:jf}

[^3]: This sentence does not include an RFC2119 modifier. Should be fixed.
[^4]: Maybe usage of CN should be deprecated here?
[^5]: Replace may with should here?
[^6]: Here more text for the requirements of TLS-PSK is needed.

## Connecting Client Identity

In RADIUS/UDP, clients are uniquely identified by their IP address.
Since the shared secret is associated with the origin IP address, if more than one RADIUS client is associated with the same IP address, then those clients also must utilize the same shared secret.
This practice is inherently insecure, as noted in {{?RFC5247}}, Section 5.3.2.

Following the different authentication modes presented in {{TLSPeerAuth}}, the identification of clients can be done by different means:

In TLS-PSK operation, a client is uniquely identified by its TLS identifier.

When using certificate fingerprints, a client is uniquely identified by the fingerprint of the presented client certificate.

When using X.509 certificates with a PKIX trust model, a client is uniquely identified by the tuple of the serial number of the presended client certificate and the issuer of the client certificate.

Note well: having identified a connecting entity does not mean the server necessarily wants to communicate with that client.
For example, if the issuer is not in a trusted set of issuers, the server may decline to perform RADIUS transactions with this client.

There are numerous trust models in PKIX environments, and it is beyond the scope of this document to define how a particular deployment determines whether a client is trustworthy.
Implementations that want to support a wide variety of trust models should expose as many details of the presented certificate to the administrator as possible so that the trust model can be implemented by the administrator.
As a suggestion, at least the following parameters of the X.509 client certificate should be exposed:

* Originating IP address
* Certificate Fingerprint
* Issuer
* Subject
* all X.509v3 Extended Key Usage
* all X.509v3 Subject Alternative Name
* all X.509v3 Certificate Policies

For TLS-PSK operation, at least the following parameters of the TLS connection should be exposed:

* Originating IP address
* TLS Identifier [^7]{:jf}

[^7]: Should this not be PSK Identifier? (The line was copied directly from RFC6614)

## RADIUS Datagrams

Authentication, Authorization, and Accounting packets are sent according to the following rules:

RADIUS/TLS clients transmit the same packet types on the connection they initiated as a RADIUS/UDP client would.
For example, they send

* Access-Request
* Accounting-Request
* Status-Server
* Disconnect-ACK
* Disconnect-NAK
* ...

RADIUS/TLS servers transmit the same packets on connections they have accepted as a RADIUS/UDP server would.
For example, they send

* Access-Challenge
* Access-Accept
* Access-Reject
* Accounting-Response
* Disconnect-Request
* ...

Due to the use of one single TCP port for all packet types, it is required that a RADIUS/TLS server signal which types of packets are supported on a server to a connecting peer.

* When an unwanted packet of type 'CoA-Request' or 'Disconnect-Request' is received, a RADIUS/TLS server needs to respond with a 'CoA-NAK' or 'Disconnect-NAK', respectively.
  The NAK SHOULD contain an attribute Error-Cause with the value 406 ("Unsupported Extension"); see {{?RFC5176}} for details.
* When an unwanted packet of type 'Accounting-Request' is received, the RADIUS/TLS server SHOULD reply with an Accounting-Response containing an Error-Cause attribute with value 406 "Unsupported Extension" as defined in {{RFC5176}}.
  A RADIUS/TLS accounting client receiving such an Accounting-Response SHOULD log the error and stop sending Accounting-Request packets to this server.

# Design Decisions

This section explains the design decisions that led to the rules defined in the previous section, as well as a reasoning behind the differences to {{RFC6614}}.

## Implications of Dynamic Peer Discovery

One mechanism to discover RADIUS-over-TLS peers dynamically via DNS is specified in {{?RFC7585}}.
While this mechanism is still under development and therefore is not a normative dependency of RADIUS/TLS, the use of dynamic discovery has potential future implications that are important to understand.

Readers of this document who are considering the deployment of DNS-based dynamic discovery are thus encouraged to read {{RFC7585}} and follow its future development.

## X.509 Certificate Considerations
{: #cert_considerations }

(1)
: If a RADIUS/TLS client is in possession of multiple certificates from different CAs (i.e., is part of multiple roaming consortia) and dynamic discovery is used, the discovery mechanism possibly does not yield sufficient information to identify the consortium uniquely (e.g., DNS discovery).
  Subsequently, the client may not know by itself which client certificate to use for the TLS handshake.
  Then, it is necessary for the server to signal to which consortium it belongs and which certificates it expects.
  If there is no risk of confusing multiple roaming consortia, providing this information in the handshake is not crucial.

(2)
: If a RADIUS/TLS server is in possession of multiple certificates from different CAs (i.e., is part of multiple roaming consortia), it will need to select one of its certificates to present to the RADIUS/TLS client.
  If the client sends the Trusted CA Indication, this hint can make the server select the appropriate certificate and prevent a handshake failure.
  Omitting this indication makes it impossible to deterministically select the right certificate in this case.
  If there is no risk of confusing multiple roaming consortia, providing this indication in the handshake is not crucial.

## Cipher Suites and Compression Negotiation Considerations

See {{RFC7525}} for considerations regarding the cipher suites and negotiation.

## RADIUS Datagram Considerations
{: #datagram_considerations }

(1)
: After the TLS session is established, RADIUS packet payloads are
  exchanged over the encrypted TLS tunnel.  In RADIUS/UDP, the
  packet size can be determined by evaluating the size of the
  datagram that arrived.  Due to the stream nature of TCP and TLS,
  this does not hold true for RADIUS/TLS packet exchange.
  Instead, packet boundaries of RADIUS packets that arrive in the
  stream are calculated by evaluating the packet's Length field.
  Special care needs to be taken on the packet sender side that
  the value of the Length field is indeed correct before sending
  it over the TLS tunnel, because incorrect packet lengths can no
  longer be detected by a differing datagram boundary.  See
  Section 2.6.4 of {{RFC6613}} for more details.

(2)
: Within RADIUS/UDP {{RFC2865}}, a shared secret is used for hiding
  attributes such as User-Password, as well as in computation of
  the Response Authenticator.  In RADIUS accounting {{RFC2866}}, the
  shared secret is used in computation of both the Request
  Authenticator and the Response Authenticator.  Since TLS
  provides integrity protection and encryption sufficient to
  substitute for RADIUS application-layer security, it is not
  necessary to configure a RADIUS shared secret.  The use of a
  fixed string for the obsolete shared secret eliminates possible
  node misconfigurations.

(3)
: RADIUS/UDP {{RFC2865}} uses different UDP ports for
  authentication, accounting, and dynamic authorization changes.
  RADIUS/TLS allocates a single port for all RADIUS packet types.
  Nevertheless, in RADIUS/TLS, the notion of a client that sends
  authentication requests and processes replies associated with
  its users' sessions and the notion of a server that receives
  requests, processes them, and sends the appropriate replies is
  to be preserved.  The normative rules about acceptable packet
  types for clients and servers mirror the packet flow behavior
  from RADIUS/UDP.

(4)
: RADIUS/UDP {{RFC2865}} uses negative ICMP responses to a newly
  allocated UDP port to signal that a peer RADIUS server does not
  support the reception and processing of the packet types in
  {{RFC5176}}.  These packet types are listed as to be received in
  RADIUS/TLS implementations.  Note well: it is not required for
  an implementation to actually process these packet types; it is
  only required that the NAK be sent as defined above.

(5)
: RADIUS/UDP {{RFC2865}} uses negative ICMP responses to a newly
  allocated UDP port to signal that a peer RADIUS server does not
  support the reception and processing of RADIUS Accounting
  packets.  There is no RADIUS datagram to signal an Accounting
  NAK.  Clients may be misconfigured for sending Accounting
  packets to a RADIUS/TLS server that does not wish to process
  their Accounting packet.  To prevent a regression of
  detectability of this situation, the Accounting-Response +
  Error-Cause signaling was introduced.

# Compatibility with Other RADIUS Transports

The IETF defines multiple alternative transports to the classic UDP transport model as defined in {{RFC2865}}, namely RADIUS over TCP {{RFC6613}}, the present document on RADIUS over TLS and RADIUS over Datagram Transport Layer Security (DTLS) {{?RFC7360}}.

RADIUS/TLS does not specify any inherent backward compatibility to RADIUS/UDP or cross compatibility to the other transports, i.e., an implementation that utilizes RADIUS/TLS only will not be able to receive or send RADIUS packet payloads over other transports.
An implementation wishing to be backward or cross compatible (i.e., wishes to serve clients using other transports than RADIUS/TLS) will need to implement these other transports along with the RADIUS/TLS transport and be prepared to send and receive on all implemented transports, which is called a "multi-stack implementation".

If a given IP device is able to receive RADIUS payloads on multiple transports, this may or may not be the same instance of software, and it may or may not serve the same purposes.
It is not safe to assume that both ports are interchangeable.
In particular, it cannot be assumed that state is maintained for the packet payloads between the transports.
Two such instances MUST be considered separate RADIUS server entities.

# Security Considerations

The computational resources to establish a TLS tunnel are significantly higher than simply sending mostly unencrypted UDP datagrams.
Therefore, clients connecting to a RADIUS/TLS node will more easily create high load conditions and a malicious client might create a Denial-of-Service attack more easily.

Some TLS cipher suites only provide integrity validation of their payload and provide no encryption.
This specification forbids the use of such cipher suites.
Since the RADIUS payload's shared secret is fixed to the well-known term "radsec", failure to comply with this requirement will expose the entire datagram payload in plaintext, including User-Password, to intermediate IP nodes.

By virtue of being based on TCP, there are several generic attack vectors to slow down or prevent the TCP connection from being established; see {{?RFC4953}} for details.
If a TCP connection is not up when a packet is to be processed, it gets re-established, so such attacks in general lead only to a minor performance degradation (the time it takes to re-establish the connection).
There is one notable exception where an attacker might create a bidding-down attack though.
If peer communication between two devices is configured for both RADIUS/TLS and RADIUS/UDP, and the RADIUS/UDP transport is the failover option if the TLS session cannot be established, a bidding-down attack can occur if an adversary can maliciously close the TCP connection or prevent it from being established.
Situtations where clients are configured in such a way are likely to occur during a migration phase from RADIUS/UDP to RADIUS/TLS.
By preventing the TLS session setup, the attacker can reduce the security of the packet payload from the selected TLS cipher suite packet encryption to the classic MD5 per-attribute encryption.
The situation should be avoided by disabling the weaker RADIUS/UDP transport as soon as the new RADIUS/TLS connection is established and tested.

RADIUS/TLS provides authentication and encryption between RADIUS peers.
In the presence of proxies, the intermediate proxies can still inspect the individual RADIUS packets, i.e., "end-to-end" encryption is not provided.
Where intermediate proxies are untrusted, it is desirable to use other RADIUS mechanisms to prevent RADIUS packet payload from inspection by such proxies.
One common method to protect passwords is the use of the Extensible Authentication Protocol (EAP) and EAP methods that utilize TLS.

When using certificate fingerprints to identify RADIUS/TLS peers, any two certificates that produce the same hash value (i.e., that have a hash collision) will be considered the same client.
Therefore, it is important to make sure that the hash function used is cryptographically uncompromised so that an attacker is very unlikely to be able to produce a hash collision with a certificate of his choice.
While this specification mandates support for SHA-1, a later revision will likely demand support for more contemporary hash functions because as of issuance of this document, there are already attacks on SHA-1.

# IANA Considerations

Upon approval, IANA should update the Reference to radsec in the Service Name and Transport Protocol Port Number Registry:

* Service Name: radsec
* Port Number: 2083
* Transport Protocol: tcp
* Description: Secure RADIUS Service
* Assignment notes: The TCP port 2083 was already previously assigned by IANA for "RadSec", an early implementation of RADIUS/TLS, prior to issuance of the experimental RFC 6614. [This document] updates RFC 6614, while maintaining backward compatibility, if configured. For further details see RFC 6614, Appendix A or [This document], {{BackwardComp}}.

--- back

# Lessons learned from deployments of the Experimental {{RFC6614}}

There are at least two major (world-scale) deployments of {{RFC6614}}.

## eduroam

eduroam is a globally operating Wi-Fi roaming consortium exclusively for persons in Research and Education. For an extensive background on eduroam and its authentication fabric architecture, refer to {{?RFC7593}}.

Over time, more than a dozen out of 100+ national branches of eduroam used RADIUS/TLS in production to secure their country-to-country RADIUS proxy connections. This number is big enough to attest that the protocol does work, and scales. The number is also low enough to wonder why RADIUS/UDP continued to be used by a majority of country deployments despite its significant security issues.

Operational experience reveals that the main reason is related to the choice of PKIX certificates for securing the proxy interconnections. Compared to shared secrets, certificates are more complex to handle in multiple dimensions:

* Lifetime: PKIX certificates have an expiry date, and need administrator attention and expertise for their renewal
* Validation: The validation of a certificate (both client and server) requires contacting a third party to verify the recovaction status. This either takes time during session setup (OCSP checks) or requires the presence of a fresh CRL on the server - this in turn requires regular update of that CRL.
* Issuance: PKIX certificates carry properties in the Subject and extensions that need to be vetted. Depending on the CA policy, a certificate request may need significant human intervention to be verified. In particular, the authorisation of a requester to operate a server for a particular NAI realm needs to be verified. This rules out public "browser-trusted" CAs; eduroam is operating a special-purpose CA for eduroam RADIUS/TLS purposes.
* Automatic failure over time: CRL refresh and certificate renewal must be attended to regularly. Failure to do so leads to failure of the authentication service. Among other reasons, employee churn with incorrectly transferred or forgotten responsibilities is a risk factor.

It appears that these complexities often outweigh the argument of improved security; and a fallback to RADIUS/UDP is seen as the more appealing option.

It can be considered an important result of the experiment in {{RFC6614}} that providing less complex ways of operating RADIUS/TLS are required. The more thoroughly specified provisions in the current document towards TLS-PSK and raw public keys are a response to this insight.

On the other hand, using RADIUS/TLS in combination with Dynamic Discovery as per {{RFC7585}} necessitates the use of PKIX certificates. So, the continued ability to operate with PKIX certificates is also important and cannot be discontinued without sacrificing vital funcionality of large roaming consortia.

## Wireless Broadband Alliance's OpenRoaming

OpenRoaming is a globally operating Wi-Fi roaming consortium for the general public, operated by the Wireless Broadband Alliance (WBA). With its (optional) settled usage of hotspots, the consortium requires both RADIUS authentication as well as RADIUS accounting.

The consortium operational procedures were defined in the late 2010s when {{RFC6614}} and {{RFC7585}} were long available. The consortium decided to fully base itself on these two RFCs.

In this architecture, using PSKs or raw public keys is not an option. The complexities around PKIX certificates as discussed in the previous section are believed to be controllable: the consortium operates its own special-purpose CA and can rely on a reliable source of truth for operator authorisation (becoming an operator requires a paid membership in WBA); expiry and revocation topics can be expected to be dealt with as high-priority because of the monetary implications in case of infrastructure failure during settled operation.

## Participating in more than one roaming consortium

It is possible for a RADIUS/TLS (home) server to participate in more than one roaming consortium, i.e. to authenticate its users to multiple clients from distinct consortia, which present client certificates from their respective consortium's CA; and which expect the server to present a certificate from the matching CA.

The eduroam consortium has chosen to cooperate with (the settlement-free parts of) OpenRoaming to allow eduroam users to log in to (settlement-free) OpenRoaming hotspots.

eduroam RADIUS/TLS servers thus may be contacted by OpenRoaming clients expecting an OpenRoaming server certificate, and by eduroam clients expecting an eduroam server certificate.

It is therefore necessary to decide on the certificate to present during TLS session establishment. To make that decision, the availability of Trusted CA Indication in the client TLS message is important.

It can be considered an important result of the experiment in {{RFC6614}} that Trusted CA Indication is an important asset for inter-connectivity of multiple roaming consortia.

# Interoperable Implementations

{{RFC6614}} is implemented and interoperates between at least three server implementations: FreeRADIUS, radsecproxy, Radiator. It is also implemented among a number of Wireless Access Points / Controllers from numerous vendors, including but not limited to: Aruba Networks, LANCOM Systems.

# Backward compatibility {#BackwardComp}

TODO describe necessary steps to configure common servers for compatibility with this version.
Hopefully the differences to {{RFC6614}} are small enough that almost no config change is necessary.

# Acknowledgments
{:numbered="false"}

Thanks to the original authors of RFC 6614: Stefan Winter, Mice McCauley, Stig Venaas and Klaas Vierenga.

TODO more acknowledgements

