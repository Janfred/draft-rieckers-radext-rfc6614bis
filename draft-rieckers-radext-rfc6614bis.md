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

normative:

informative:


--- abstract

This document specifies a transport profile for RADIUS using Transport Layer Security (TLS) over TCP as the transport protocol.
This enables dynamic trust relationships between RADIUS servers as well as encrypting RADIUS traffic between servers using a shared secret.

--- middle

# Introduction

TODO Introduction


## Conventions and Definitions

{::boilerplate bcp14-tagged}

## Changes from RFC6614

# Security Considerations

TODO Security


# IANA Considerations

TODO IANA considerations
Upon approval, IANA should update the Reference to radsec in the Service Name and Transport Protocol Port Number Registry:

* Service Name: radsec
* Port Number: 2083
* Transport Protocol: tcp
* Description: Secure RADIUS Service
* Assignment notes: The TCP port 2083 was already previously assigned by IANA for "RadSec", an early implementation of RADIUS/TLS, prior to issuance of the experimental RFC 6614. [This document] updates RFC 6614, while maintaining backward compatibility, if configured. For further details see RFC 6614, Appendix A or [This document], {{BackwardComp}}.

--- back

# Backward compatibility {#BackwardComp}

TODO describe necessary steps to configure common servers for compatibility with this doc.

# Acknowledgments
{:numbered="false"}

Thanks to the original authors of RFC 6614: Stefan Winter, Mice McCauley, Stig Venaas and Klaas Vierenga.

TODO acknowledge.

