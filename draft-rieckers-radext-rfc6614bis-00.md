---
title: "Transport Layer Security (TLS) Encryption for RADIUS"
abbrev: "RADIUS over TLS"
category: std

obsoletes: 6614

docname: draft-rieckers-radext-rfc6614bis-00
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


informative:
  eduroam:
    author:
      org: Trans-European Research and Education Networking Association
    title: "eduroam Hompage"
    date: 2007
    format:
      TXT: https://www.eduroam.org/
  MD5-attacks: DOI.10.1007/11799313_17
  radsec-whitepaper:
    author:
      org: Open Systems Consultants
    title: "RadSec - a secure, reliable RADIUS Protocol"
    date: 2005-05
    format:
      TXT: http://www.open.com.au/radiator/radsec-whitepaper.pdf

--- abstract

This document specifies a transport profile for RADIUS using Transport Layer Security (TLS) over TCP as the transport protocol.
This enables dynamic trust relationships between RADIUS servers.

--- middle

# Introduction

The RADIUS protocol {{!RFC2865}} is a widely deployed authentication and authorization protocol.
The supplementary RADIUS Accounting specification {{!RFC2866}} provides accounting mechanisms, thus delivering a full Authentication, Authorization, and Accounting (AAA) solution.
However, RADIUS is experiencing several shortcomings, such as its dependency on the unreliable transport protocol UDP and the lack of security for large parts of its packet payload.
RADIUS security is based on the MD5 algorithm, which has been proven to be insecure.

The main focus of RADIUS over TLS is to provide a means to secure the communication between RADIUS/TCP peers using TLS.
The most important use of this specification lies in roaming environments where RADIUS packets need to be transferred through different administrative domains and untrusted, potentially hostile networks.
An example for a worldwide roaming environment that uses RADIUS over TLS to secure communication is "eduroam", see {{eduroam}}.

There are multiple known attacks on the MD5 algorithm that is used in RADIUS to provide integrity protection and a limited confidentiality protection (see {{MD5-attacks}}).
RADIUS over TLS wraps the entire RADIUS packet payload into a TLS stream and thus mitigates the risk of attacks on MD5.

Because of the static trust establishment between RADIUS peers (IP address and shared secret), the only scalable way of creating a massive deployment of RADIUS servers under the control of different administrative entities is to introduce some form of a proxy chain to route the access requests to their home server.
This creates a lot of overhead in terms of possible points of failure, longer transmission times, as well as middleboxes through which authentication traffic flows.
These middleboxes may learn privacy-relevant data while forwarding requests.
The new features in RADIUS over TLS obsolete the use of IP addresses and shared MD5 secrets to identify other peers and thus allow the use of more contemporary trust models, e.g., checking a certificate by inspecting the issuer and other certificate properties.

## Requirements Language

{::boilerplate bcp14-tagged}

## Terminology

RADIUS/TLS node:
: a RADIUS-over-TLS client or server

RADIUS/TLS Client:
: a RADIUS-over-TLS instance that initiates a new connection.

RADIUS/TLS Server:
: a RADIUS-over-TLS instance that listens on a RADIUS-over-TLS port and accepts new connections

RADIUS/UDP:
: a classic RADIUS transport over UDP as defined in {{RFC2865}}

## Document Status

This document is an Experimental RFC.

   It is one out of several approaches to address known cryptographic
   weaknesses of the RADIUS protocol (see also {{compatibility}}).
   The
   specification does not fulfill all recommendations on a AAA transport
   profile as per {{?RFC3539}}; in particular, by being based on TCP as a
   transport layer, it does not prevent head-of-line blocking issues.

   If this specification is indeed selected for advancement to Standards
   Track, certificate verification options ({{connection_setup}}, point 2) need
   to be refined.

   Another experimental characteristic of this specification is the
   question of key management between RADIUS/TLS peers.  RADIUS/UDP only
   allowed for manual key management, i.e., distribution of a shared
   secret between a client and a server.  RADIUS/TLS allows manual
   distribution of long-term proofs of peer identity as well (by using
   TLS-PSK ciphersuites, or identifying clients by a certificate
   fingerprint), but as a new feature enables use of X.509 certificates
   in a PKIX infrastructure.  It remains to be seen if one of these
   methods will prevail or if both will find their place in real-life
   deployments.  The authors can imagine pre-shared keys (PSK) to be
   popular in small-scale deployments (Small Office, Home Office (SOHO)
   or isolated enterprise deployments) where scalability is not an issue
   and the deployment of a Certification Authority (CA) is considered
   too much of a hassle; however, the authors can also imagine large
   roaming consortia to make use of PKIX.  Readers of this specification
   are encouraged to read the discussion of key management issues within
   {{?RFC6421}} as well as {{?RFC4107}}.

It has yet to be decided whether this approach is to be chosen for
   Standards Track.  One key aspect to judge whether the approach is
   usable on a large scale is by observing the uptake, usability, and
   operational behavior of the protocol in large-scale, real-life
   deployments.

   An example for a worldwide roaming environment that uses RADIUS over
   TLS to secure communication is "eduroam", see {{eduroam}}.

# Normative: Transport Layer Security for RADIUS/TCP

## TCP port and Packet Types

The default destination port number for RADIUS over TLS is TCP/2083.
There are no separate ports for authentication, accounting, and dynamic authorization changes.
The source port is arbitrary.
See {{datagram_considerations}} for considerations regarding the separation of authentication, accounting, and dynamic authorization traffic.

## TLS Negotiation

RADIUS/TLS has no notion of negotiating TLS in an established connection.
Servers and clients need to be preconfigured to use RADIUS/TLS for a given endpoint.


## Connection Setup
{: #connection_setup }

RADIUS/TLS nodes

  1. establish TCP connections as per {{!RFC6613}}.  Failure to connect
       leads to continuous retries, with exponentially growing intervals
       between every try.  If multiple servers are defined, the node MAY
       attempt to establish a connection to these other servers in
       parallel, in order to implement quick failover.
  2. after completing the TCP handshake, immediately negotiate TLS
       sessions according to {{!RFC5246}} or its predecessor TLS 1.1.  The
       following restrictions apply:
      * Support for TLS v1.1 {{?RFC4346}} or later (e.g., TLS 1.2
          {{!RFC5246}}) is REQUIRED.  To prevent known attacks on TLS
          versions prior to 1.1, implementations MUST NOT negotiate TLS
          versions prior to 1.1.
      * Support for certificate-based mutual authentication is
          REQUIRED.
      * Negotiation of mutual authentication is REQUIRED.
      * Negotiation of a ciphersuite providing for confidentiality as
          well as integrity protection is REQUIRED.  Failure to comply
          with this requirement can lead to severe security problems,
          like user passwords being recoverable by third parties.  See
          {{security_considerations}} for details.
      * Support for and negotiation of compression is OPTIONAL.
      * Support for TLS-PSK mutual authentication {{!RFC4279}} is
          OPTIONAL.
      * RADIUS/TLS implementations MUST, at a minimum, support
          negotiation of the TLS_RSA_WITH_3DES_EDE_CBC_SHA, and SHOULD
          support TLS_RSA_WITH_RC4_128_SHA and
          TLS_RSA_WITH_AES_128_CBC_SHA as well (see {{ciphersuite_considerations}}.
      * In addition, RADIUS/TLS implementations MUST support
          negotiation of the mandatory-to-implement ciphersuites
          required by the versions of TLS that they support.
  3. Peer authentication can be performed in any of the following
       three operation models:
      * TLS with X.509 certificates using PKIX trust models (this
          model is mandatory to implement):
          * Implementations MUST allow the configuration of a list of
             trusted Certification Authorities for incoming connections.
          * Certificate validation MUST include the verification rules
             as per {{!RFC5280}}.
          * Implementations SHOULD indicate their trusted Certification
             Authorities (CAs).  For TLS 1.2, this is done using
             {{!RFC5246}}, Section 7.4.4, "certificate_authorities" (server
             side) and {{!RFC6066}}, Section 6 "Trusted CA Indication"
             (client side).  See also {{cert_considerations}}.
          * Peer validation always includes a check on whether the
             locally configured expected DNS name or IP address of the
             server that is contacted matches its presented certificate.
             DNS names and IP addresses can be contained in the Common
             Name (CN) or subjectAltName entries.  For verification,
             only one of these entries is to be considered.  The
             following precedence applies: for DNS name validation,
             subjectAltName:DNS has precedence over CN; for IP address
             validation, subjectAltName:iPAddr has precedence over CN.
             Implementors of this specification are advised to read
             {{?RFC6125}}, Section 6, for more details on DNS name
             validation.
          * Implementations MAY allow the configuration of a set of
             additional properties of the certificate to check for a
             peer's authorization to communicate (e.g., a set of allowed
             values in subjectAltName:URI or a set of allowed X509v3
             Certificate Policies).
          * When the configured trust base changes (e.g., removal of a
             CA from the list of trusted CAs; issuance of a new CRL for
             a given CA), implementations MAY renegotiate the TLS
             session to reassess the connecting peer's continued
             authorization.
      * TLS with X.509 certificates using certificate fingerprints
          (this model is optional to implement): Implementations SHOULD
          allow the configuration of a list of trusted certificates,
          identified via fingerprint of the DER encoded certificate
          octets.  Implementations MUST support SHA-1 as the hash
          algorithm for the fingerprint.  To prevent attacks based on
          hash collisions, support for a more contemporary hash function
          such as SHA-256 is RECOMMENDED.
      * TLS using TLS-PSK (this model is optional to implement).
  4. start exchanging RADIUS datagrams (note {{datagram_considerations}} (1)).  The
       shared secret to compute the (obsolete) MD5 integrity checks and
       attribute encryption MUST be "radsec" (see {{datagram_considerations}} (2)).

## Connecting Client Identity
{: #connecting_client_id }

In RADIUS/UDP, clients are uniquely identified by their IP address.
Since the shared secret is associated with the origin IP address, if more than one RADIUS client is associated with the same IP address, then those clients also must utilize the same shared secret, a practice that is inherently insecure, as noted in {{!RFC5247}}.

RADIUS/TLS supports multiple operation modes.

In TLS-PSK operation, a client is uniquely identified by its TLS identifier.

In TLS-X.509 mode using fingerprints, a client is uniquely identified by the fingerprint of the presented client certificate.

In TLS-X.509 mode using PKIX trust models, a client is uniquely identified by the tuple (serial number of presented client certificate;Issuer).

Note well: having identified a connecting entity does not mean the server necessarily wants to communicate with that client.
For example, if the Issuer is not in a trusted set of Issuers, the server may decline to perform RADIUS transactions with this client.

There are numerous trust models in PKIX environments, and it is beyond the scope of this document to define how a particular deployment determines whether a client is trustworthy.
Implementations that want to support a wide variety of trust models should expose as many details of the presented certificate to the administrator as possible so that the trust model can be implemented by the administrator.
As a suggestion, at least the following parameters of the X.509 client certificate should be exposed:

* Originating IP address
* Certificate Fingerprint
* Issuer
* Subject
* all X509v3 Extended Key Usage
* all X509v3 Subject Alternative Name
* all X509v3 Certificate Policies

In TLS-PSK operation, at least the following parameters of the TLS connection should be exposed:

* Originating IP address
* TLS Identifier

## RADIUS Datagrams
{: #radius_datagrams }

Authentication, Authorization, and Accounting packets are sent according to the following rules:

RADIUS/TLS clients transmit the same packet types on the connection they initiated as a RADIUS/UDP client would (see {{datagram_considerations}} (3) and (4)). 
For example, they send

* Access-Request
* Accounting-Request
* Status-Server
* Disconnect-ACK
* Disconnect-NAK
* ...

and they receive

* Access-Accept
* Accounting-Response
* Disconnect-Request
* ...

RADIUS/TLS servers transmit the same packet types on connections they have accepted as a RADIUS/UDP server would.
For example, they send

* Access-Challenge
* Access-Accept
* Access-Reject
* Accounting-Response
* Disconnect-Request
* ...

and they receive

* Access-Request
* Accounting-Request
* Status-Server
* Disconnect-ACK
* ...

Due to the use of one single TCP port for all packet types, it is required that a RADIUS/TLS server signal which types of packets are supported on a server to a connecting peer.
See also {{datagram_considerations}} for a discussion of signaling.

* When an unwanted packet of type 'CoA-Request' or 'Disconnect-Request' is received, a RADIUS/TLS server needs to respond with a 'CoA-NAK' or 'Disconnect-NAK', respectively.
  The NAK SHOULD contain an attribute Error-Cause with the value 406 ("Unsupported Extension"); see {{?RFC5176}} for details.
* When an unwanted packet of type 'Accounting-Request' is received, the RADIUS/TLS server SHOULD reply with an Accounting-Response containing an Error-Cause attribute with value 406 "Unsupported Extension" as defined in {{RFC5176}}.
  A RADIUS/TLS accounting client receiving such an Accounting-Response SHOULD log the error and stop sending Accounting-Request packets.

# Informative: Design decisions
{: #design_decisions }

This section explains the design decisions that led to the rules defined in the previous section.

## Implications of Dynamic Peer Discovery
{: #dynamic_peer_discovery }

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

## Ciphersuites and Compression Negotiation Considerations
{: #ciphersuite_considerations }

Not all TLS ciphersuites in {{RFC5246}} are supported by available TLS
   tool kits, and licenses may be required in some cases.  The existing
   implementations of RADIUS/TLS use OpenSSL as a cryptographic backend,
   which supports all of the ciphersuites listed in the rules in the
   normative section.

   The TLS ciphersuite TLS_RSA_WITH_3DES_EDE_CBC_SHA is mandatory to
   implement according to {{RFC4346}}; thus, it has to be supported by
   RADIUS/TLS nodes.

   The two other ciphersuites in the normative section are widely
   implemented in TLS tool kits and are considered good practice to
   implement.


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
{: #compatibility }

The IETF defines multiple alternative transports to the classic UDP transport model as defined in {{RFC2865}}, namely RADIUS over TCP {{RFC6613}} and the present document on RADIUS over TLS.  The IETF also
proposed RADIUS over Datagram Transport Layer Security (DTLS)
{{?RFC7360}}.

RADIUS/TLS does not specify any inherent backward compatibility to RADIUS/UDP or cross compatibility to the other transports, i.e., an implementation that utilizes RADIUS/TLS only will not be able to receive or send RADIUS packet payloads over other transports.
An implementation wishing to be backward or cross compatible (i.e., wishes to serve clients using other transports than RADIUS/TLS) will need to implement these other transports along with the RADIUS/TLS transport and be prepared to send and receive on all implemented transports, which is called a "multi-stack implementation".

If a given IP device is able to receive RADIUS payloads on multiple transports, this may or may not be the same instance of software, and it may or may not serve the same purposes.
It is not safe to assume that both ports are interchangeable.
In particular, it cannot be assumed that state is maintained for the packet payloads between the transports.
Two such instances MUST be considered separate RADIUS server entities.


# Diameter Compatibility
{: #diameter_comp }

Since RADIUS/TLS is only a new transport profile for RADIUS, the
compatibility of RADIUS/TLS - Diameter {{?RFC3588}} and RADIUS/UDP
{{RFC2865}} - Diameter {{RFC3588}} is identical.  The considerations
regarding payload size in {{RFC6613}} apply.


# Security Considerations
{: #security_considerations }

The computational resources to establish a TLS tunnel are significantly higher than simply sending mostly unencrypted UDP datagrams.
Therefore, clients connecting to a RADIUS/TLS node will more easily create high load conditions and a malicious client might create a Denial-of-Service attack more easily.

Some TLS ciphersuites only provide integrity validation of their payload, and provide no encryption.
This specification forbids the use of such ciphersuites.
Since the RADIUS payload's shared secret is fixed to the well-known term "radsec" (see {{connection_setup}} (4)), failure to comply with this requirement will expose the entire datagram payload in plaintext, including User-Password, to intermediate IP nodes.

By virtue of being based on TCP, there are several generic attack vectors to slow down or prevent the TCP connection from being established; see {{?RFC4953}} for details.
If a TCP connection is not up when a packet is to be processed, it gets re-established, so such attacks in general lead only to a minor performance degradation (the time it takes to re-establish the connection).
There is one notable exception where an attacker might create a bidding-down attack though.
If peer communication between two devices is configured for both RADIUS/TLS (i.e., TLS security over TCP as a transport, shared secret fixed to "radsec") and RADIUS/UDP (i.e., shared secret security with a secret manually configured by the administrator), and the RADIUS/UDP transport is the failover option if the TLS session cannot be established, a bidding-down attack can occur if an adversary can maliciously close the TCP connection or prevent it from being established.
Situations where clients are configured in such a way are likely to occur during a migration phase from RADIUS/UDP to RADIUS/TLS.
By preventing the TLS session setup, the attacker can reduce the security of the packet payload from the selected TLS ciphersuite packet encryption to the classic MD5 per-attribute encryption.
The situation should be avoided by disabling the weaker RADIUS/UDP transport as soon as the new RADIUS/TLS connection is established and tested.
Disabling can happen at either the RADIUS client or server side:

  * Client side: de-configure the failover setup, leaving RADIUS/TLS
      as the only communication option
  *  Server side: de-configure the RADIUS/UDP client from the list of
      valid RADIUS clients

RADIUS/TLS provides authentication and encryption between RADIUS peers.
In the presence of proxies, the intermediate proxies can still inspect the individual RADIUS packets, i.e., "end-to-end" encryption is not provided.
Where intermediate proxies are untrusted, it is desirable to use other RADIUS mechanisms to prevent RADIUS packet payload from inspection by such proxies.
One common method to protect passwords is the use of the Extensible Authentication Protocol (EAP) and EAP methods that utilize TLS.

When using certificate fingerprints to identify RADIUS/TLS peers, any two certificates that produce the same hash value (i.e., that have a hash collision) will be considered the same client.
Therefore, it is important to make sure that the hash function used is cryptographically uncompromised so that an attacker is very unlikely to be able to produce a hash collision with a certificate of his choice.
While this specification mandates support for SHA-1, a later revision will likely demand support for more contemporary hash functions because as of issuance of this document, there are already attacks on SHA-1.

# IANA Considerations
{: #IANA_considerations }

No new RADIUS attributes or packet codes are defined.  IANA has
   updated the already assigned TCP port number 2083 to reflect the
   following:
* Reference: {{?RFC6614}}
* Assignment Notes: The TCP port 2083 was already previously
  assigned by IANA for "RadSec", an early implementation of RADIUS/
  TLS, prior to issuance of this RFC.  This early implementation can
  be configured to be compatible to RADIUS/TLS as specified by the
  IETF.  See RFC 6614, Appendix A for details.


# Acknowledgements

RADIUS/TLS was first implemented as "RADSec" by Open Systems
 Consultants, Currumbin Waters, Australia, for their "Radiator" RADIUS
 server product (see {{radsec-whitepaper}}).

 Funding and input for the development of this document was provided
 by the European Commission co-funded project "GEANT2" [TODO: outdated reference] and
 further feedback was provided by the TERENA Task Force on Mobility
 and Network Middleware [TODO: outdated reference].


--- back

# Implementation Overview: Radsec


Radiator implements the RadSec protocol for proxying requests with
the \<Authby RADSEC\> and \<ServerRADSEC\> clauses in the Radiator
configuration file.

The \<AuthBy RADSEC\> clause defines a RadSec client, and causes
Radiator to send RADIUS requests to the configured RadSec server
using the RadSec protocol.

The \<ServerRADSEC\> clause defines a RadSec server, and causes
Radiator to listen on the configured port and address(es) for
connections from \<Authby RADSEC\> clients.  When an \<Authby RADSEC\>
client connects to a \<ServerRADSEC\> server, the client sends RADIUS
requests through the stream to the server.  The server then handles
the request in the same way as if the request had been received from
a conventional UDP RADIUS client.

Radiator is compliant to RADIUS/TLS if the following options are
used:

   \<AuthBy RADSEC\>

   *  Protocol tcp

   *  UseTLS

   *  TLS_CertificateFile

   *  Secret radsec

   \<ServerRADSEC\>

   *  Protocol tcp

   *  UseTLS

   *  TLS_RequireClientCert

   *  Secret radsec

As of Radiator 3.15, the default shared secret for RadSec connections
is configurable and defaults to "mysecret" (without quotes).  For
compliance with this document, this setting needs to be configured
for the shared secret "radsec".  The implementation uses TCP
keepalive socket options, but does not send Status-Server packets.
Once established, TLS connections are kept open throughout the server
instance lifetime.

# Implementation Overview: radsecproxy


The RADIUS proxy named radsecproxy was written in order to allow use
of RadSec in current RADIUS deployments.  This is a generic proxy
that supports any number and combination of clients and servers,
supporting RADIUS over UDP and RadSec.  The main idea is that it can
be used on the same host as a non-RadSec client or server to ensure
RadSec is used on the wire; however, as a generic proxy, it can be
used in other circumstances as well.

The configuration file consists of client and server clauses, where
there is one such clause for each client or server.  In such a
clause, one specifies either "type tls" or "type udp" for TLS or UDP
transport.  Versions prior to 1.6 used "mysecret" as a default shared
secret for RADIUS/TLS; version 1.6 and onwards uses "radsec".  For
backwards compatibility with older versions, the secret can be
changed (which makes the configuration not compliant with this
specification).

In order to use TLS for clients and/or servers, one must also specify
where to locate CA certificates, as well as certificate and key for
the client or server.  This is done in a TLS clause.  There may be
one or several TLS clauses.  A client or server clause may reference
a particular TLS clause, or just use a default one.  One use for
multiple TLS clauses may be to present one certificate to clients and
another to servers.

If any RadSec (TLS) clients are configured, the proxy will, at
startup, listen on port 2083, as assigned by IANA for the OSC RadSec
implementation.  An alternative port may be specified.  When a client
connects, the client certificate will be verified, including checking
that the configured Fully Qualified Domain Name (FQDN) or IP address
matches what is in the certificate.  Requests coming from a RadSec
client are treated exactly like requests from UDP clients.

At startup, the proxy will try to establish a TLS connection to each
(if any) of the configured RadSec (TLS) servers.  If it fails to
connect to a server, it will retry regularly.  There is some back-off
where it will retry quickly at first, and with longer intervals
later.  If a connection to a server goes down, it will also start
retrying regularly.  When setting up the TLS connection, the server
certificate will be verified, including checking that the configured
FQDN or IP address matches what is in the certificate.  Requests are
sent to a RadSec server, just like they would be to a UDP server.

The proxy supports Status-Server messages.  They are only sent to a
server if enabled for that particular server.  Status-Server requests
are always responded to.

This RadSec implementation has been successfully tested together with
Radiator.  It is a freely available, open-source implementation.  For
source code and documentation, see [TODO outdated reference].
