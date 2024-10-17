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
normative:
informative:

--- abstract

Proxying IP in HTTP allows building a VPN through HTTP load balancers. However,
at the time of writing, that mechanism doesn't offer a mechanism for
communicating DNS configuration information inline. In contrast, most existing
VPN protocols provide a mechanism to exchange DNS configuration information.
This document describes an extension that exchanges this information using HTTP
capsules.

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
Remote Access VPN (see {{Section 8.1 of CONNECT-IP}}), but not for cases like
IP Flow Forwarding (see {{Section 8.3 of CONNECT-IP}}).

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
to request DNS configuration information and to assign it. Similarly, this
mechanism is bidirectional: either endpoint can request DNS configuration
information by sending a DNS_REQUEST capsule, and either endpoint can send DNS
configuration information in a DNS_ASSIGN capsule. These capsules follow the
format defined below.

~~~
Nameserver {
  Type (i),
  Length (i),
  Value (...),
}
~~~
{: #ns-format title="Nameserver Format"}

Each Nameserver structure contains the following fields:

Type:

: An integer representing the protocol over which DNS queries and responses are
sent. See below for possible values. Encoded as a variable-length integer.

Length:

: The length of the following Value field, encoded as a variable-length integer.

Value:

: Nameserver configuration value, depends on the Type. This is commonly an IP
address, but for other protocols it can also represent a URI template or a
domain name.

This document defines the following types:

* DNS over port 53. Type = 0. DNS is sent unencrypted over UDP or TCP port 53,
  as per {{!DNS=RFC1035}}. The Value is an IP address (either IPv4 or IPv6)
  encoded in network byte order. Length SHALL be either 32 or 128 bits.

* DNS over TLS. Type = 1. DNS is sent over TLS, as per {{!DoT=RFC7858}}. The
  Value is a domain name, optionally followed by a colon and a port. The
  encoding is the same as an authority without userinfo as defined in {{Section
  3.2 of !URI=RFC3986}}. It is encoded as ASCII, and not null-terminated. IPv4
  and IPv6 addresses can be encoded using this format, though IPv6 addresses
  need to be enclosed in square brackets.

* DNS over QUIC. Type = 2. DNS is sent over QUIC, as per {{!DoQ=RFC9250}}. The
  Value is a domain name, encoded the same as for DNS over TLS.

* DNS over HTTPS. Type = 3. DNS is sent over HTTPS, as per {{!DoH=RFC8484}}.
  The Value is a URI Template. It is encoded as ASCII, and not null-terminated.

* TODO: properly define an IANA registry with GREASE for future types.

~~~
Domain {
  Domain Length (i),
  Domain Name (..),
}
~~~
{: #domain-format title="Internal Domain Format"}

Each Domain contains the following fields:

Domain Length:

: Length of the following Domain field, encoded as a variable-length integer.

Domain Name:

: Fully Qualified Domain Name in DNS presentation format and using an
Internationalized Domain Names for Applications (IDNA) A-label
({{!IDNA=RFC5890}}).

~~~
DNS Configuration {
  Request ID (i),
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

Request ID:

: Request identifier, encoded as a variable-length integer. If this DNS
Configuration is part of a request, then this contains a unique request
identifier. If this DNS configuration is part of an assignment that is in
response to a DNS configuration request then this field SHALL contain the value
of the corresponding field in the request. If this DNS configuration is part of
an unsolicited assignment, this field SHALL be zero.

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

## DNS_REQUEST Capsule {#dns-req}

The DNS_REQUEST capsule (see {{iana}} for the value of the capsule type) allows
an endpoint to request DNS configuration from its peer. The capsule allows the
endpoint to optionally indicate a preference for which DNS configuration it
would get assigned. The sender can indicate that it has no preference by not
sending any nameservers or domain names in its request DNS Configuration.

~~~
DNS_REQUEST Capsule {
  Type (i) = DNS_REQUEST,
  Length (i),
  DNS Configuration (..),
}
~~~
{: #dns-req-format title="DNS_REQUEST Capsule Format"}

When sending a DNS_REQUEST capsule, the sender MUST generate and send a new
non-zero request ID that was not previously used on this IP Proxying stream.
Note that this request ID namespace is distinct from the one used by
ADDRESS_ASSIGN capsules (see {{Section 4.7.1 of CONNECT-IP}}).

An endpoint that receives a DNS_REQUEST capsule SHALL reply by sending a
DNS_ASSIGN capsule with the corresponding request ID. That DNS_ASSIGN capsule
MAY be empty, that indicates that its sender has no DNS configuration to share
with its peer.

## DNS_ASSIGN Capsule {#dns-assign}

The DNS_ASSIGN capsule (see {{iana}} for the value of the capsule type) allows
an endpoint to send DNS configuration to its peer.

~~~
DNS_ASSIGN Capsule {
  Type (i) = DNS_ASSIGN,
  Length (i),
  DNS Configuration (..),
}
~~~
{: #dns-assign-format title="DNS_ASSIGN Capsule Format"}

When sending a DNS_ASSIGN capsule in response to a received DNS_REQUEST
capsule, the Request ID field in the DNS_ASSIGN capsule SHALL be set to the
value in the received DNS_REQUEST capsule. Otherwise the request ID MUST be set
to zero.

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
requests over the same HTTPS connection.

# Examples

## Full-Tunnel Consumer VPN

A full-tunnel consumer VPN hosted at masque.example could configure the client
to use DNS over HTTPS to the IP proxy itself by sending the following
configuration.

~~~
DNS Configuration = {
  Nameservers = [{
    Type = 3,  // DNS over HTTPS
    Value = "https://masque.example/dns-query{?dns}",
  }],
  Internal Domains = [""],
  Search Domains = [],
}
~~~
{: #ex-doh title="Full Tunnel Example"}

## Split-Tunnel Enterprise VPN

An enterprise switching their preexisting IPsec split-tunnel VPN could use the
following configuration.

~~~
DNS Configuration = {
  Nameservers = [{
    Type = 0,  // DNS over 53
    Value = 2001:db8::1,
  }, {
    Type = 0,  // DNS over 53
    Value = 192.0.2.33,
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

The requirement for an endpoint to always send DNS_ASSIGN capsules in response
to DNS_REQUEST capsules could lead it to buffer unbounded amounts of memory if
the underlying stream is blocked by flow or congestion control. Implementations
MUST place an upper bound on that buffering and abort the stream if that limit
is reached.

# IANA Considerations {#iana}

This document, if approved, will request IANA add the following values to the
"HTTP Capsule Types" registry maintained at
<[](https://www.iana.org/assignments/masque)>.

|   Value    | Capsule Type |
|:-----------|:-------------|
| 0x1460B736 |  DNS_ASSIGN  |
| 0x1460B737 |  DNS_REQUEST |
{: #iana-capsules-table title="New Capsules"}

Note that, if this document is approved, the values defined above will be
replaced by smaller ones before publication.

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

The mechanism in this document was inspired by {{IKEv2}} and
{{?IKEv2-DNS=RFC8598}}. The author would like to thank {{{Alex
Chernyakhovsky}}} and {{{Tommy Pauly}}} for their contributions.
