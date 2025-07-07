---
title: "DNS Configuration for Proxying IP in HTTP"
abbrev: "CONNECT-IP DNS"
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
at the time of writing, that mechanism doesn't offer a mechanism for
communicating DNS configuration information inline. In contrast, most existing
VPN protocols provide a mechanism to exchange DNS configuration information.
This document describes an extension that exchanges this information using HTTP
capsules. This mechanism supports encrypted DNS transports.

--- middle

# Introduction

Proxying IP in HTTP ({{!CONNECT-IP=RFC9484}}) allows building a VPN through
HTTP load balancers. However, at the time of writing, that mechanism doesn't
offer a mechanism for communicating DNS configuration information inline. In
contrast, most existing VPN protocols provide a mechanism to exchange DNS
configuration information (e.g., {{?IKEv2=RFC7296}}). This document describes
an extension that exchanges this information using HTTP capsules
({{!HTTP-DGRAM=RFC9297}}). This document does not define any new ways to convey
DNS queries or responses, only a mechanism to exchange DNS configuration
information.

Note that this extension is meant for cases where connect-ip is used like a
Remote Access VPN (see {{Section 8.1 of CONNECT-IP}}) or Site-to-Site VPN
(see {{Section 8.2 of CONNECT-IP}}), but not for cases like IP Flow Forwarding
(see {{Section 8.3 of CONNECT-IP}}).

This specification uses Service Bindings ({{!SVCB=RFC9460}}) to exchange
information about nameservers, such as which encrypted DNS transport is
supported. This allows support for DNS over HTTPS ({{!DoH=RFC8484}}), DNS over
QUIC ({{!DoQ=RFC9250}}), DNS over TLS ({{!DoT=RFC7858}}), unencrypted DNS over
UDP port 53 and TCP port 53 ({{!DNS=RFC1035}}), as well as potential future DNS
transports.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses terminology from {{!QUIC=RFC9000}}. Where this document
defines protocol types, the definition format uses the notation from {{Section
1.3 of QUIC}}. This specification uses the variable-length integer encoding
from {{Section 16 of !QUIC=RFC9000}}. Variable-length integer values do not
need to be encoded in the minimum number of bytes necessary.

In this document, we use the term "nameserver" to refer to a DNS recursive
resolver as defined in {{Section 6 of !DNS-TERMS=RFC8499}}, and the term
"domain name" is used as defined in {{Section 2 of !DNS-TERMS}}.

# Mechanism

Similar to how Proxying IP in HTTP exchanges IP address configuration
information ({{Section 4.7 of CONNECT-IP}}), this mechanism leverages capsules
to signal DNS configuration information. Similarly, this mechanism is
bidirectional: either endpoint can send DNS configuration information in a
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

DNS_ASSIGN capsule MAY include multiple DNS configurations if different DNS servers
are responsible for separate internal domains.

If multiple DNS_ASSIGN capsules are sent in one direction, each DNS_ASSIGN capsule
supersedes prior ones.

# Handling

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

# Examples

## Full-Tunnel Consumer VPN

A full-tunnel consumer VPN hosted at masque.example.org could configure the client
to use DNS over HTTPS to the IP proxy itself by sending the following
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

## Split-Tunnel Enterprise VPN

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

This document, if approved, will request IANA add the following value to the
"HTTP Capsule Types" registry maintained at
<[](https://www.iana.org/assignments/masque)>.

Value:

: 0x1ACE79EC (if this document is approved, the values defined above will be
replaced by smaller ones before publication)

Capsule Type:

: DNS_ASSIGN

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
