---
title: "DNS and PREF64 Configuration for Proxying IP in HTTP"
abbrev: "CONNECT-IP DNS and PREF64"
category: std
docname: draft-ietf-masque-connect-ip-dns-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
wg: MASQUE
venue:
  group: "Multiplexed Application Substrate over QUIC Encryption"
  type: "Working Group"
  mail: "masque@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/masque/"
  github: "ietf-wg-masque/draft-ietf-masque-connect-ip-dns"
  latest: "https://ietf-wg-masque.github.io/draft-ietf-masque-connect-ip-dns/draft-ietf-masque-connect-ip-dns.html"
keyword:
  - quic
  - http
  - datagram
  - udp
  - tcp
  - proxy
  - tunnels
  - quic in quic
  - turtles all the way down
  - masque
  - http-ng
  - dns
author:
  -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    region: CA
    code: 94043
    country: United States of America
    email: dschinazi.ietf@gmail.com
  -
    ins: Y. Rosomakho
    name: Yaroslav Rosomakho
    org: Zscaler
    street: 120 Holger Way
    city: San Jose
    region: CA
    code: 95134
    country: United States of America
    email: yrosomakho@zscaler.com
normative:
informative:

--- abstract

Proxying IP in HTTP allows building a VPN through HTTP load balancers. However,
at the time of writing, that mechanism lacks a way to communicate network
configuration details such as DNS information or IPv6 NAT64 prefixes (PREF64).
In contrast, most existing VPN protocols provide mechanisms to exchange
such configuration information.

This document defines extensions that convey DNS and PREF64 configuration
using HTTP capsules. This mechanism supports encrypted DNS transports
and can be used to inform peers about network translation prefixes for IPv6/IPv4
synthesis.

--- middle

# Introduction

Proxying IP in HTTP ({{!CONNECT-IP=RFC9484}}) allows building a VPN through
HTTP load balancers. However, at the time of writing, that mechanism lacks
a way to communicate network configuration details such as DNS information
or IPv6 NAT64 prefixes (PREF64). In contrast, most existing VPN protocols provide
mechanisms to exchange such configuration information (for example
{{?IKEv2=RFC7296}} supports discovery of DNS servers).

This document defines extensions that allow CONNECT-IP peers to convey DNS
and PREF64 configuration information to its peer using HTTP capsules
({{!HTTP-DGRAM=RFC9297}}). These extensions do not define any new way to
transport DNS queries or responses, but instead enable the exchange of DNS
resolver configuration and NAT64 prefix information inline with CONNECT-IP
sessions.

Note that these extensions are meant for cases where CONNECT-IP operates as a
Remote Access VPN (see {{Section 8.1 of CONNECT-IP}}) or a Site-to-Site VPN
(see {{Section 8.2 of CONNECT-IP}}), but not for cases like IP Flow Forwarding
(see {{Section 8.3 of CONNECT-IP}}).

For DNS configuration, this specification uses Service Bindings ({{!SVCB=RFC9460}})
to exchange information about nameservers, such as which encrypted DNS transport is
supported. This allows support for DNS over HTTPS ({{!DoH=RFC8484}}), DNS over
QUIC ({{!DoQ=RFC9250}}), DNS over TLS ({{!DoT=RFC7858}}), unencrypted DNS over
UDP port 53 and TCP port 53 ({{!DNS=RFC1035}}), as well as potential future DNS
transports. PREF64 configuration is represented in a way similar to Router
Advertisement option defined in {{Section 4 of ?PREF64-RA=RFC8781}}.

Similar to how Proxying IP in HTTP exchanges IP address configuration
information ({{Section 4.7 of CONNECT-IP}}), the mechanism defined in this document
leverages capsules. Similarly, these mechanisms are bidirectional: either endpoint
can send configuration information.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology from {{!QUIC=RFC9000}}. Where this document
defines protocol types, the definition format uses the notation from {{Section
1.3 of QUIC}}. This specification uses the variable-length integer encoding
from {{Section 16 of !QUIC=RFC9000}}. Variable-length integer values do not
need to be encoded in the minimum number of bytes necessary.

In this document, we use the term "nameserver" to refer to a DNS recursive
resolver as defined in {{Section 6 of !DNS-TERMS=RFC8499}}, and the term
"domain name" is used as defined in {{Section 2 of !DNS-TERMS}}.

# DNS Configuration Mechanism

Either endpoint can send DNS configuration information in a
`DNS_ASSIGN` capsule. The capsule format is defined below.

## Domain Structure

~~~
Domain {
  Domain Length (i),
  Domain Name (..),
}
~~~
{: #domain-format title="Domain Format"}

Each Domain contains the following fields:

Domain Length:

: Length of the following Domain field, encoded as a variable-length integer.

Domain Name:

: Fully Qualified Domain Name in DNS presentation format and using an
Internationalized Domain Names for Applications (IDNA) A-label
({{!IDNA=RFC5890}}).

## Nameserver Structure {#domain-struct}

~~~
Nameserver {
  Service Priority (16),
  IPv4 Address Count (i),
  IPv4 Address (32) ...,
  IPv6 Address Count (i),
  IPv6 Address (128) ...,
  Authentication Domain Name (..),
  Service Parameters Length (i),
  Service Parameters (..),
}
~~~
{: #ns-format title="Nameserver Format"}

Each Nameserver structure contains the following fields:

Service Priority:

: The priority of this attribute compared to other nameservers, as specified in
{{Section 2.4.1 of SVCB}}. Since this specification relies on using Service
Bindings in ServiceMode ({{Section 2.4.3 of SVCB}}), this field MUST NOT be set
to 0.

IPv4 Address Count:

: The number of IPv4 Address fields following this field. Encoded as a
variable-length integer.

IPv4 Address:

: Sequence of IPv4 Addresses that can be used to reach this nameserver. Encoded
in network byte order.

IPv6 Address Count:

: The number of IPv6 Address fields following this field. Encoded as a
variable-length integer.

IPv6 Address:

: Sequence of IPv6 Addresses that can be used to reach this nameserver. Encoded
in network byte order.

Authentication Domain Name:

: A Domain structure (see {{domain-struct}}) representing the domain name of
the nameserver. This MAY be empty if the nameserver only supports unencrypted
DNS (as traditionally sent over UDP port 53 and TCP port 53).

Service Parameters Length:

: Length of the following Service Parameters field, encoded as a
variable-length integer.

Service Parameters:

: Set of service parameters that apply to this nameserver. Encoded using the
wire format specified in {{Section 2.2 of SVCB}}.

Service parameters allow exchanging additional information about the nameserver:

* The "port" service parameter is used to indicate which port the nameserver is
  reachable on. If no "port" service parameter is included, this indicates that
  default port numbers should be used.

* The "alpn" service parameter is used to indicate which encrypted DNS
  transports are supported by this nameserver. If the "no-default-alpn" service
  parameter is omitted, that indicates that the nameserver supports unencrypted
  DNS, as is traditionally sent over UDP port 53 and TCP port 53. In that case,
  the sum of IPv4 Address Count and IPv6 Address Count MUST be nonzero. If
  Authentication Domain Name is empty, the "alpn" and "no-default-alpn" service
  parameter MUST be omitted.

* The "dohpath" service parameter is used to convey a relative DNS over HTTPS
  URI Template, see {{Section 5 of !SVCB-DNS=RFC9461}}.

* The service parameters MUST NOT include "ipv4hint" or "ipv6hint" SvcParams,
  as they are superseded by the included IP addresses.

## DNS Configuration Structure

~~~
DNS Configuration {
  Nameserver Count (i),
  Nameserver (..) ...,
  Internal Domain Count (i),
  Internal Domain (..) ...,
  Search Domain Count (i),
  Search Domain (..) ...,
}
~~~
{: #assigned-addr-format title="Assigned Address Format"}

Each DNS Configuration contains the following fields:

Nameserver Count:

: The number of Nameserver structures following this field. Encoded as
a variable-length integer.

Nameserver:

: A series of Nameserver structures representing nameservers.

Internal Domain Count:

: The number of Domain structures following this field. Encoded as a
variable-length integer.

Internal Domain:

: A series of Domain structures representing internal domain names.

Search Domain Count:

: The number of Domain structures following this field. Encoded as a
variable-length integer.

Search Domain:

: A series of Domain structures representing search domains.
{:newline="true" spacing="normal"}

## DNS_ASSIGN Capsule {#dns-assign}

The DNS_ASSIGN capsule (see {{iana}} for the value of the capsule type) allows
an endpoint to send DNS configuration to its peer.

~~~
DNS_ASSIGN Capsule {
  Type (i) = DNS_ASSIGN,
  Length (i),
  DNS Configuration (..) ...,
}
~~~
{: #dns-assign-format title="DNS_ASSIGN Capsule Format"}

DNS_ASSIGN capsule MAY include multiple DNS configurations if different DNS
servers are responsible for separate internal domains.

If multiple DNS_ASSIGN capsules are sent in one direction, each DNS_ASSIGN
capsule supersedes prior ones.

## Handling

Note that internal domains include subdomains. In other words, if the DNS
configuration contains a domain, that indicates that the corresponding domain
and all of its subdomains can be resolved by the nameservers exchanged in the
same DNS configuration. Sending an empty string as an internal domain indicates
the DNS root; i.e., that the corresponding nameserver can resolve all domain
names.

As with other IP Proxying capsules, the receiver can decide whether to use or
ignore the configuration information. For example, in the consumer VPN
scenario, clients will trust the IP proxy and apply received DNS configuration,
whereas IP proxies will ignore any DNS configuration sent by the client.

If the IP proxy sends a DNS_ASSIGN capsule containing a DNS over HTTPS
nameserver, then the client can validate whether the IP proxy is authoritative
for the origin of the URI template. If this validation succeeds, the client
SHOULD send its DNS queries to that nameserver directly as independent HTTPS
requests. When possible, those requests SHOULD be coalesced over the same HTTPS
connection.

## Examples

### Full-Tunnel Consumer VPN

A full-tunnel consumer VPN hosted at masque.example.org could configure the
client to use DNS over HTTPS to the IP proxy itself by sending the following
configuration.

~~~
DNS Configuration = {
  Nameservers = [{
    Service Priority = 1,
    IPv4 Address = [],
    IPv6 Address = [],
    Authentication Domain Name = "masque.example.org",
    Service Parameters = {
      alpn=h2,h3
      dohpath=/dns-query{?dns}
    },
  }],
  Internal Domains = [""],
  Search Domains = [],
}
~~~
{: #ex-doh title="Full Tunnel Example"}

### Split-Tunnel Enterprise VPN

An enterprise switching their preexisting IKEv2/IPsec split-tunnel VPN to
connect-ip could use the following configuration.

~~~
DNS Configuration = {
  Nameservers = [{
    Service Priority = 1,
    IPv4 Address = [192.0.2.33],
    IPv6 Address = [2001:db8::1],
    Authentication Domain Name = "",
    Service Parameters = {},
  }],
  Internal Domains = ["internal.corp.example"],
  Search Domains = [
    "internal.corp.example",
    "corp.example",
  ],
}
~~~
{: #ex-split title="Split Tunnel Example"}

# PREF64 Configuration Mechanism

This document defines a new capsule type, `PREF64`, that allows a CONNECT-IP peer
to communicate the IPv6 NAT64 prefixes to be used for IPv6/IPv4 address synthesis.
This information enables an endpoint operating in an IPv6-only environment to
construct IPv4-reachable addresses from IPv6 literals when a NAT64 translator is in use.

## PREF64 Capsule {#pref64-capsule}

Each PREF64 capsule conveys zero or more NAT64 prefixes. If multiple capsules are sent
in the same direction, the most recent one replaces any previously advertised prefixes.
Empty PREF64 capsule is used to inform that NAT64 prefixes are not available.

The capsule has the following structure (see {{iana}} for the value of the capsule type):

~~~
PREF64 Capsule {
  Type (i) = PREF64,
  Length (i),
  NAT64 Address Format (104) ...,
}
~~~
{: #pref64-format title="PREF64 Capsule Format"}

Individual NAT64 address format has the following structure:

~~~
NAT64 Prefix {
  Prefix Length (8),
  IPv6 Address Format (96),
}
~~~
{: #nat64-prefix title="NAT64 Address Format"}

NAT64 prefix fields are:

Prefix Length:

: The length of the NAT64 prefix in bits encoded as a single byte. Valid values are 32, 40, 48,
56, 64, and 96. These lengths correspond to the prefix sizes permitted for the IPv6-to-IPv4
address synthesis algorithm described in {{Section 2.2 of !IPv6-TRANSLATOR=RFC6052}}.

IPv6 Address Format:

: Address prefix and suffix separated by 8 zero bits as defined in {{Section 2.2 of IPv6-TRANSLATOR}}.
Note that this field is always 96 bits long, regardless of the value in the Prefix Length field
preceding it. If Prefix Length is set to 96, the IPv6 Address Format consists of the prefix without a
separator or a suffix.

## Handling

Upon receiving a PREF64 capsule, a peer updates its local NAT64 configuration for the
corresponding CONNECT-IP session. The newly received PREF64 overrides any previously
received PREF64 in the same direction.

If an endpoint receives a capsule that does not meet one of the requirements listed in {{pref64-capsule}}, or
with a length that is not a multiple of 13 bytes, it MUST treat it as malformed. An empty
PREF64 capsule invalidates any previously received NAT64 Address Formats.

## Example

A CONNECT-IP peer would use the following capsule to signal a single NAT64 prefix of `64:ff9b::/96`

~~~
PREF64 Capsule {
  Type (i) = PREF64,
  Length (i) = 13,
  NAT64 Prefix {
    Prefix Length (8) = 96
    IPv6 Address Format (96) = 0x00 0x64 0xff 0x9b 0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00,
  }
}
~~~
{: #ex-pref64 title="PREF64 Capsule Example"}

# Security Considerations

Acting on received DNS_ASSIGN capsules can have significant impact on endpoint
security. Endpoints MUST ignore DNS_ASSIGN capsules unless it has reason to
trust its peer and is expecting DNS configuration from it.

This mechanism can cause an endpoint to use a nameserver that is outside of the
connect-ip tunnel. While this is acceptable in some scenarios, in others it
could break the privacy properties provided by the tunnel. To avoid this,
implementations need to ensure that DNS_ASSIGN capsules are not sent before the
corresponding ROUTE_ADVERTISEMENT capsule.

# IANA Considerations {#iana}

This document, if approved, will request IANA add the following values to the
"HTTP Capsule Types" registry maintained at
<[](https://www.iana.org/assignments/masque)>.

| Value | Capsule Type |
| --- | --- |
| 0x1ACE79EC (if this document is approved, this value will be
replaced by a smaller one before publication) | DNS_ASSIGN |
| 0x274C0FBC (if this document is approved, this value will be
replaced by a smaller one before publication) | PREF64 |


All of these new entries use the following values for these fields:

Status:

: provisional (permanent if this document is approved)

Reference:

: This document

Change Controller:

: IETF

Contact:

: masque@ietf.org

Notes:

: None
{: spacing="compact" newline="false"}

--- back

# Acknowledgments
{:numbered="false"}

The mechanism in this document was inspired by {{IKEv2}},
{{?IKEv2-DNS=RFC8598}}, and {{?IKEv2-SVCB=RFC9464}}. The authors would like to
thank {{{Alex Chernyakhovsky}}}, {{{Tommy Pauly}}}, and other enthusiasts in
the MASQUE Working Group for their contributions.
