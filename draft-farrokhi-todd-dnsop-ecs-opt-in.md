---
title: "Client Opt-In Signaling for EDNS Client Subnet"
abbrev: "ECS Client Opt-In"
docname: draft-farrokhi-todd-dnsop-ecs-opt-in-latest
category: exp
stream: IETF
ipr: trust200902
area: Internet
keyword:
  - DNS
  - EDNS
  - ECS
  - EDNS Client Subnet
  - privacy
  - opt-in
pi: [toc, tocindent, sort refs, symrefs, strict, compact, inline]
author:
  - ins: B. Farrokhi
    name: Babak Farrokhi
    org: Quad9
    country: Netherlands
    email: babak@farrokhi.net
  - ins: J. Todd
    name: John Todd
    org: Quad9
    country: United States of America
    email: jtodd@loligo.com

--- abstract

EDNS Client Subnet (ECS) lets a recursive resolver send part of a
client's network address to authoritative servers, which use it to tailor
their answers.  Whether a resolver does this is determined by its
configuration and cannot be chosen per query. An operator who wants to
offer both tailored answers and address privacy runs two resolvers on
separate addresses and relies on clients to choose the right one.  RFC
7871 lets a client opt out of a default that forwards its address
information.  Asking instead for a shorter prefix requires the client to
supply the address those bits are taken from, which a client behind a NAT
or a VPN does not know.

This document defines an opt-in mechanism. A client asks the recursive
resolver to forward its address information by including an EDNS(0)
option in its query, and may use that option to limit how many address
bits the resolver forwards. A resolver that implements this document
forwards nothing for a client that does not send the option. A resolver
that has not implemented it ignores the option and behaves as it does
today. An operator
can therefore run a single resolver on one address for both the clients
that want tailored responses and the clients that want their addresses
withheld. Each client chooses per query. The resolver reports in its
response how many address bits it will forward.

--- middle

# Introduction

EDNS Client Subnet {{!RFC7871}} lets a recursive resolver send a prefix
of the client's network address to authoritative servers, which use it to
tailor their answers.

A resolver either uses ECS or it does not, based on how it is configured. A client of a resolver with ECS enabled can opt out by sending an ECS option with a SOURCE PREFIX-LENGTH of 0, which {{Section 7.1.2 of !RFC7871}} obliges the recursive resolver to honor. That client can also ask for a shorter prefix, but {{Section 6 of !RFC7871}} requires the ECS option to contain the address those bits are taken from, and a client behind a NAT or a VPN does not know the address the resolver sees. A client that sends nothing has its address information forwarded.

An operator who wants to offer ECS to some clients and withhold it from others runs a second resolver on a second address and tells users which one to use.  That increases the operational burden, resource usage and complexity. It also hands the privacy decision to whoever configures the client's resolver address, which is usually not the user whose address is forwarded.

This document defines an opt-in mechanism.  A recursive resolver that implements it forwards a client's address information only for a query that includes the opt-in option, and that option can also limit how many address bits the resolver forwards.  The resolver reports in its response how many it applied.


## Scope

This document defines the signal and the resolver behavior it requires.
It does not change {{!RFC7871}}, and it does not define any new
behavior for authoritative servers.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

DNS terminology follows BCP 219 {{!RFC9499}}.  ECS terminology
follows {{!RFC7871}} and is not redefined here.  This includes SOURCE
PREFIX-LENGTH, SCOPE PREFIX-LENGTH, Tailored Response, Intermediate
Nameserver, and Forwarding Resolver, none of which {{!RFC9499}} defines.

The term "signaling client" means a client whose query included the option
defined in this document and for which the effective prefix length
defined in {{resolver-behavior}} is nonzero.

# The ECS Opt-In Option

## Option Format {#option-format}

This document specifies a new EDNS(0) {{!RFC6891}} option by which a
client asks the resolver to forward its address information.

OPTION-CODE for this option is TBD.

OPTION-LENGTH MUST have a value of 0 or 1 for queries and MUST have the
value 1 for responses.

OPTION-DATA, where present, is a single field:

~~~
                 0   1   2   3   4   5   6   7
               +---+---+---+---+---+---+---+---+
               |         PREFIX-LENGTH         |
               +---+---+---+---+---+---+---+---+
~~~
{: title="OPTION-DATA Format for the ECS Opt-In Option"}

PREFIX-LENGTH is an unsigned 1-octet field.  In a query it is the largest
number of address bits the client permits the resolver to forward.  A
query that omits OPTION-DATA sets no such limit.  In a response it is the effective prefix
length defined in {{resolver-behavior}}, the largest number of address
bits the resolver will forward for that query.

The option has no address family field.  The resolver interprets
PREFIX-LENGTH in the address family of the address it forwards.

## Client Behavior

A client that wants to permit the resolver to forward a prefix of its address
MUST include exactly one instance of this option in the query.  To limit
how many address bits the resolver forwards, it uses the one-octet form.
With the zero-length form the resolver's own maximum applies, as
{{resolver-behavior}} describes.

A client that permits nothing to be forwarded SHOULD omit the option.  A
PREFIX-LENGTH of 0 forwards nothing either, but on an unencrypted
transport it tells anyone on the path that the client implements this
specification.

A client MAY include an ECS option in the same query.  A client that
knows the public address it is seen as coming from can supply it that
way.  A client that does not know that address can still set a limit with the
one-octet form, which does not need one.
{{relationship-to-ecs}} specifies how the two options interact.

A client MUST treat this option as absent in a response whose
OPTION-LENGTH is not 1, and in a response that includes more than one
instance of it.  Neither case tells the client what the resolver
forwarded.

A client MUST NOT treat the arrival of a response as evidence that its
limit was honored.  A resolver that does not implement this option
ignores it, per {{Section 6.1.2 of !RFC6891}}, and answers as it would
have anyway.  {{security-considerations}} covers how much a client
can trust the reported value.

## Recursive Resolver Behavior {#resolver-behavior}

A resolver implementing this document MUST NOT forward a client's
address information for a query that does not include this option.

For a query that includes this option, the resolver selects the address
to forward.  That address is the source address of the query, so a client
need not know the address the resolver observes.  Where the query
included an ECS option, the resolver MAY instead take FAMILY and ADDRESS
from that option, which {{Section 7.1.1 of !RFC7871}} permits.  That case arises
with a query from a Forwarding Resolver, where the source address is the
forwarder's own and not its client's.

The resolver then calculates an effective prefix length, the smallest of

* the PREFIX-LENGTH in the query, if it used the one-octet form,
* the SOURCE PREFIX-LENGTH of an ECS option, if the query included one,
* the resolver's own maximum cacheable prefix length, and
* the maximum prefix length of the address family of the selected
  address, that is 32 for IPv4 and 128 for IPv6.

The resolver MUST NOT forward more bits of that address than the
effective prefix length.  {{Section 7.1.1 of !RFC7871}} already combines
two of these four the same way, having the resolver use the shorter of
the incoming SOURCE PREFIX-LENGTH and its own maximum cacheable prefix
length.

A resolver MAY decline to forward a client's address information for a
query, a choice {{Section 5 of !RFC7871}} leaves with the resolver.  The
effective prefix length for such a query is 0.

If the effective prefix length is nonzero, the resolver constructs an
ordinary ECS option as described in {{Section 7.1.1 of !RFC7871}}, with
FAMILY and ADDRESS from the selected address and SOURCE PREFIX-LENGTH set
to the effective prefix length.  The rest of the exchange, including how
the resolver handles the authoritative server's SCOPE PREFIX-LENGTH,
follows {{!RFC7871}} unchanged.

The resolver MUST treat the query as though this option were absent, and
therefore forward nothing, in each of the following cases:

* OPTION-LENGTH is greater than 1
* more than one instance of the option is present

This departs from the usual handling of a malformed EDNS option.
{{Section 6 of !RFC7871}} recommends FORMERR for a malformed ECS option,
and {{?RFC7873}} and {{?RFC9660}} specify FORMERR for their own options.
This document does not, because the option is an optional privacy
control.  A client that encodes it incorrectly would lose name resolution
altogether, whereas treating the option as absent forwards nothing and
lets resolution continue.

A PREFIX-LENGTH larger than the family maximum is not an error.  The
calculation above reduces it to the family maximum.

### Response

If the query included this option, the response MUST include exactly one
instance of it, with PREFIX-LENGTH set to the effective prefix length.

If the query did not include this option, or the resolver treated it as
absent under {{resolver-behavior}}, the response MUST NOT include this
option.  The resolver forwarded no address information for that query and
therefore did not use ECS, so the response requirements of
{{Section 7.2.2 of !RFC7871}} do not apply, as {{relationship-to-ecs}}
explains.  The client receives the response it
would have received from a resolver that does not implement ECS.

If the query included an ECS option and the effective prefix length is
nonzero, the response MUST include an ECS option, as {{Section 7.2.2 of
!RFC7871}} requires of a resolver that uses ECS.  Its FAMILY, ADDRESS and
SOURCE PREFIX-LENGTH are those of the query, which {{Section 11.2 of
!RFC7871}} requires as a countermeasure against birthday attacks.  That
requirement holds even where the resolver forwarded fewer address bits
upstream than the query contained.  SCOPE PREFIX-LENGTH reports the scope
of the answer.  Otherwise the response MUST NOT include an ECS option.

The reported value is the effective prefix length computed for that
query, and it does not depend on whether the answer came from cache.

A client that sent the option can distinguish three cases from the
response.  An absent option means the resolver does not implement this
specification, or that the option was removed in transit.  Such a
resolver may still use ECS of its own accord.  A value of 0 means the
resolver implements this specification and will forward nothing.  A value
of N means the resolver will forward at most N bits of that client's
address.

## Relationship to the ECS Option {#relationship-to-ecs}

The two options have different roles, and a query may include either or
both.  This option asks the resolver to forward the client's address
information.  An ECS option supplies the address to forward and, in its
SOURCE PREFIX-LENGTH, a limit of its own.

An ECS option cannot state a limit without an address.
{{Section 6 of !RFC7871}} requires ADDRESS to hold the client's address
truncated to SOURCE PREFIX-LENGTH bits, and a stub resolver behind a NAT,
a carrier-grade NAT, or a VPN may not know its actual outbound address that is exposed to the resolver.  The private address it does know is no substitute, because
{{Section 7.5 of !RFC7871}} counts private and unroutable address space
among the reasons to refuse the query.  Such a client sets its limit with
the one-octet form of this option that does not need an address.

When both options state a limit, the smaller governs, as
{{resolver-behavior}} specifies.  A larger value in this option cannot
raise the ECS limit, because {{Section 7.1.2 of !RFC7871}} forbids an
Intermediate Nameserver to send more bits of client address than the
originating query included.

An ECS option on its own is not an opt-in.  For a query that includes one
and no instance of the opt-in option, the resolver forwards nothing, as
{{resolver-behavior}} requires.  A Forwarding Resolver may add an ECS
option for the clients it serves, as {{Section 7.1.3 of !RFC7871}}
describes, so an ECS option says nothing about what the client that sent
the query wants.  A resolver never forwards this option
({{non-transitivity}}), so only that client can have added it.

{{Section 7.1.1 of !RFC7871}} and {{Section 7.5 of !RFC7871}} tell a
resolver to return REFUSED when it will not use a client's ECS option.  A
resolver implementing this document answers the query instead.  It MUST
NOT refuse a query solely because the query included an ECS option
without this option, nor solely because it used the address it observed
in place of the one the client supplied.  {{Section 7.5 of !RFC7871}}
already makes an exception of this kind, noting "that a query MUST NOT be
refused solely because it provides 0 address bits".

{{Section 7.2.2 of !RFC7871}} states its response requirements for the
case "When an Intermediate Nameserver uses ECS".  This document reads
that condition per query, so the requirements do not apply to a query for
which the resolver forwards nothing.

Each value below is further limited by the maximum prefix length of the
address family, and a resolver that declines under {{resolver-behavior}}
forwards nothing in any of these cases.

| Query includes | Bits of the client's address forwarded |
|---|---|
| neither option | none |
| an ECS option only | none |
| this option, zero-length | the resolver's maximum |
| this option, one octet N | the smallest of N and the resolver's maximum |
| both options | the smallest of all limits present |
{: title="Address Bits Forwarded for Each Combination of Options"}

## Non-Transitivity {#non-transitivity}

A resolver MUST NOT include this option in queries it sends to
authoritative servers, and MUST NOT copy it from a client query into any
upstream query.

{{Section 1 of !RFC6891}} describes EDNS as "a hop-by-hop extension to
DNS", negotiated between each pair of hosts in the resolution process,
and {{Section 6.1.1 of !RFC6891}} forbids forwarding an OPT RR.  This
option is an instruction to the recursive resolver that receives the
query.  It means nothing to an authoritative server, which learns the
client's network from an ECS option.

A Forwarding Resolver is itself a client of the resolver it queries.  It
MAY include this option in its own upstream queries, and MUST NOT set a
limit larger than the one its client set.
{{Section 7.1.3 of !RFC7871}} places the same duty on a Forwarding
Resolver using ECS, which must honor the SOURCE PREFIX-LENGTH
restrictions in the incoming query from its client.

A Forwarding Resolver that constructs the ECS option itself reports to
its client the number of address bits it forwards.  Where it leaves the
ECS option to the resolver it queries, it MUST NOT report more than the
value in the upstream response, and MUST report 0 if that response
includes no instance of this option.

# Caching {#caching}

A resolver MUST NOT return an answer obtained using ECS to a client that
is not a signaling client, and MUST record for each cache entry whether
it obtained that entry using ECS.

{{Section 7.3.2 of !RFC7871}} matches a lookup from a client that sent no
ECS option against that client's own address, which could return a
Tailored Response fetched for someone else.  Excluding entries obtained
using ECS leaves no matching network, and that section then requires the
resolver to perform resolution as usual, which it explains is "to avoid
Tailored Responses in the cache from being returned to the wrong
clients".

For signaling clients, the caching and longest-prefix matching rules of
{{Section 7.3 of !RFC7871}} apply unchanged.  A cache entry whose SOURCE
PREFIX-LENGTH is longer than a signaling client's effective prefix length
MAY answer that client, because serving it forwards nothing.  A client's
limit applies only to the address information the resolver forwards and
thus does not restrict which cache entry may answer that client.

Keeping signaling and non-signaling clients on separate caches implements
the rule above, and a resolver that already supports more than one cache
needs no new machinery for it.

# Design Rationale

This document defines a new EDNS(0) option code instead of encoding the
signal in the ECS option, because every such encoding collides with a
rule in {{!RFC7871}}.  {{alternatives}} records the encodings considered.

A resolver that does not implement this option ignores it and behaves as
it does today, because {{Section 6.1.2 of !RFC6891}} requires that "Any
OPTION-CODE values not understood by a responder or requestor MUST be
ignored".

A client can request forwarding and set a limit without supplying an
address.  The REFUSED guidance in {{Section 7.1.1 of !RFC7871}} and
{{Section 7.5 of !RFC7871}}, and the requirement in
{{Section 11.2 of !RFC7871}} that a response mirror the ECS fields of the
query, apply only to a client-supplied address and thus do not cover such
a query.

An option with different lengths in queries and responses has precedent.
{{?RFC7314}} defines a zero-length EDNS EXPIRE option in a query and a
four-octet one in a response, and {{?RFC7828}} keeps its TIMEOUT field
out of queries and in responses.

The resolver side needs no new permission.
{{Section 7.1.1 of !RFC7871}} already lets a resolver use the address it
observed in place of one the client supplied.  This document adds the
client-side signal that asks for it.

# Deployment Considerations

A client can discover whether a resolver implements this option by
sending it and looking at the response, which costs one query and is
described in {{resolver-behavior}}.  That is sufficient for most
purposes.

A resolver that does not implement this option ignores it and may forward
the client's address information anyway, including for the query used as
a probe.  A client probing a resolver it has not used before SHOULD
include an ECS option with SOURCE PREFIX-LENGTH 0.
{{Section 7.1.2 of !RFC7871}} obliges a resolver that uses ECS to honor
that opt-out, and a resolver implementing this option takes the smallest
limit present, so neither forwards anything.  The response still shows
whether the option is supported, and a second query without the ECS
option reveals the resolver's own maximum.

Where a resolver publishes information about itself using DNS Resolver
Information {{?RFC9606}}, support for this option would be a natural
thing to advertise there.  Defining such a key is left to future work,
since the response already gives clients an easy way to find out.

# Security Considerations {#security-considerations}

The value in a response is the resolver's own statement about what it
forwarded, and DNSSEC does not protect it, since OPT records are not
signed ({{Section 9 of !RFC7871}}).  An encrypted transport keeps that
value from being altered in transit and does not make it accurate.  A
resolver can forward more address bits than the client permitted and
report a smaller number, which the client can detect only by observing
what authoritative servers receive.  This option is a request to a
resolver that the client has already trusted with every query it sends,
and it gives no protection against a resolver that is hostile or
compromised.

On an unencrypted transport, an on-path attacker can strip this option,
add it, or change its value.  Stripping it leaves the resolver forwarding
nothing.  Adding it, or raising the value, forwards address bits the
client did not offer.  The same attacker can change the value in the
response, so a client on an unencrypted transport cannot rely on it.  A
client that needs its limit respected SHOULD use an encrypted transport
such as {{?RFC7858}}, {{?RFC8484}}, or {{?RFC9250}}.

A client that remembers whether a resolver supports this option is
trusting a single response, which an on-path attacker can alter or forge.
Such a client SHOULD confirm support over an encrypted transport.

An attacker can send this option from forged source addresses, which
subjects a resolver that would otherwise forward nothing to the cache
pollution described in {{Section 11.3 of !RFC7871}}.

A resolver that implements this option and serves signaling and
non-signaling clients from one cache returns tailored answers to clients
that did not opt in, and no field in the response marks an answer as
tailored.  The requirement in {{caching}} is therefore part of
implementing this option.

# Privacy Considerations {#privacy-considerations}

Under {{!RFC7871}}, a client of an ECS-enabled resolver has its address
information forwarded unless it opts out.  Under this document, a
resolver forwards nothing unless the client asks.

For a client that opts in, the analysis in {{Section 11 of !RFC7871}}
applies unchanged, with one addition that the client can set the limit
without supplying an address, as {{relationship-to-ecs}} describes.  The
forwarded prefix reaches every authoritative server to which the resolver
sends ECS, along with anyone able to observe that traffic.
{{Section 12.1 of !RFC7871}} bounds that set, recommending that a
resolver omit the option toward nameservers that did not return it and
never send it to root, top-level and effective top-level domain servers.
A client
should set the shortest limit that still yields useful answers, and
should not treat a short prefix as anonymous.

On an unencrypted transport, anyone on the path can see that a query
includes this option, which marks the client as one that implements this
document.  The one-octet form also reveals the client's limit.  Neither
identifies a client on its own.

# IANA Considerations

IANA is requested to assign an option code from the "DNS EDNS0 Option
Codes (OPT)" registry in the "Domain Name System (DNS) Parameters"
registry group.  Values in the range 1 to 65000 in that registry are
assigned under the Expert Review policy.

| Value | Name | Status | Reference |
| TBD | edns-client-subnet-opt-in | Optional | This document |

--- back

# Alternatives Considered {#alternatives}

This appendix records the encodings considered and not adopted.

A reserved address used as a sentinel, for example an ECS option
containing 127.0.0.1/32 that a resolver would read as an instruction to
substitute the source address it observed.  A resolver acting on it must
either echo the sentinel, which {{Section 7.5 of !RFC7871}} warns yields
a response that appears tailored for the network named in the query when
it is tailored for another, or echo the address it substituted and fail
the check in {{Section 11.2 of !RFC7871}}, which has a client discard a
response whose FAMILY, ADDRESS, and SOURCE PREFIX-LENGTH do not match its
query.  That check is also a cache-poisoning defense, since an off-path
attacker must match those fields for a forged response to be accepted.

Handling of such an address is unspecified in any case.
{{Section 11.3 of !RFC7871}} has a resolver treat an unroutable address
as equivalent to its own identity, and {{Section 7.5 of !RFC7871}} lists
private and unroutable address space among the reasons to return REFUSED.
At the time of writing, one large public resolver answered a query
containing 203.0.113.0/24 with REFUSED while another forwarded the same
value to the authoritative server verbatim.

A short ADDRESS field.  {{Section 6 of !RFC7871}} requires ADDRESS to
hold exactly the octets needed for SOURCE PREFIX-LENGTH bits and
recommends FORMERR for a server that receives too few or too many.  An
empty ADDRESS is valid only with SOURCE PREFIX-LENGTH 0, which already
means opt-out.

The SCOPE PREFIX-LENGTH field as an extension point.
{{Section 6 of !RFC7871}} requires it to be set to 0 in queries.

A novel FAMILY value.  {{Section 7.1.2 of !RFC7871}} notes that "at least
one major authoritative server will ignore the option if FAMILY is not 1
or 2", so such a value might degrade quietly, but that behavior is
unspecified, and the IANA Address Family Numbers registry is not the
place to record a DNS signaling convention.

An EDNS header flag bit.  The EDNS Header Flags registry holds few bits
and assigns them under Standards Action, which does not fit an
experimental client preference.

A bare opt-in with no one-octet form.  The zero-length form defined in
{{option-format}} serves most clients, but on its own it leaves a client
that wants a limit nowhere to put one, because an ECS option requires an
address that client may not have, as {{relationship-to-ecs}} describes.

A wider payload with a family selector and a limit per family.  A client
cannot predict the family it will be seen as coming from, and the extra
fields add nothing beyond sending a different value per family.

The closest prior art is EDNS-Client-Tag and EDNS-Server-Tag
{{?I-D.bellis-dnsop-edns-tags}}, an opaque client-to-server signal with a
response counterpart, which has the same shape as this design.  That work
was not completed.

# Open Issues

This section records decisions that are provisional in this revision.  It
will be removed before publication.

Whether a response to a query that included this option should always
include it, as {{resolver-behavior}} requires, or should omit it where
the resolver forwards nothing.  Always including it lets a client tell a
resolver that declined from one that does not implement the option, at
the cost of one octet in every such response.

Whether the reported value should be the effective prefix length, as
specified here, or the number of address bits the resolver forwarded
while answering that query.  The two differ when the answer comes from
cache.

Whether a malformed option should be treated as absent, as
{{resolver-behavior}} specifies, or should draw a FORMERR, which
{{?RFC9660}} requires for its own option and
{{Section 6 of !RFC7871}} recommends for a malformed ECS option.

Whether the condition in {{Section 7.2.2 of !RFC7871}}, "When an
Intermediate Nameserver uses ECS", is read per query or per server.
{{relationship-to-ecs}} reads it per query, which lets a resolver answer
a query that includes an ECS option without returning one.

The registry name requested in the IANA Considerations and the
human-readable name of the option are provisional.
