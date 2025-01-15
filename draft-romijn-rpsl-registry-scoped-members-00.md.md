---
stand_alone: true
ipr: trust200902
cat: info
submissiontype: IETF
area: 
wg:

docname: draft-romijn-rpsl-registry-scoped-members-00
updates: 2622, 4012

title: Registry scoped members for RPSL set objects
abbrev: Registry scoped members for RPSL set objects
lang: en
kw:
  - rpsl
author:
- ins: S. Romijn
  name: Sasha Romijn
  org: Reliably Coded
  city: Amsterdam
  country: NL
  email: sasha@reliablycoded.nl


normative:
  RFC2622:
  RFC4012:
informative:

--- abstract

This document updates {{RFC2622}} and {{RFC4012}} by specifying `src-members`,
a new attribute on as-set and route-set
objects in the Routing Policy Specification Language (RPSL).
This attribute allows references between sets to include a specific
registry name, avoiding problematic ambiguity while resolving.
A new validation rule allows gradual upgrades and backwards compatibility.

--- middle

# Introduction

The Routing Policy Specification Language (RPSL) {{RFC2622}} defines the
as-set and route-set objects, extended in {{RFC4012}}.
These are among the most common objects in Internet Routing Registry (IRR) registries.
These sets often reference other sets as well through their `members` and `mp-members`
attributes {{RFC4012}}. Server and client software can follow these references to 
resolve a set down to its members, a set of prefixes or ASes.

However\:

1. There are multiple large IRR registries (and often, many are loaded in one IRR server).
1. Sets often reference sets in other registries.
1. There is no uniqueness in set names amongst the different registries.
1. Hence, multiple objects may exist in the same server with the same name, making references to them ambiguous.

There is no current way to prevent such ambiguity, both for operators who create the legitimate objects
and those who try to resolve them.

While not defined in the standard,
most IRRs allow scoping a set name to an ASN by using single colons, e.g. `AS65000:AS-EXAMPLE`, possibly
with authorization required through aut-num AS65000.
However, this authorization only works within a single IRR registry.
It only helps for some accidental conflicts.

To solve this, this documents adds `src-members` to as-set and route-set objects,
using a IRR registry name prefix with a double colon.
For example, "RIPE::AS-EXAMPLE", a format already used by operators informally.
Continued availability of existing `(mp-)members` attributes
together with new validation rules, ensures backwards compatibility.

## Requirements Language

{::boilerplate bcp14-tagged}

# The `src-members` attribute

## as-set class

The new `src-members` attribute on as-set is similar to the `members` attribute
from {{RFC2622}}, except that the AS set name MUST be prefixed with a registry name.

{:vspace}
Attribute:
: `src-members`

{:vspace}
Value:
: list of ([`as-number`] or [`registry-name`]::[`as-set-name`])

{:vspace}
Type:
: optional, multi-valued


## route-set class

The new `src-members` attribute on as-set is similar to the `mp-members` attribute 
from {{RFC4012}}, except that any references to set names MUST 
be prefixed with a registry name.

{:vspace}
Attribute:
: `src-members`

{:vspace}
Value:
: list of ([`ipv4-address-prefix-range`] or [`ipv6-address-prefix-range`] or [`registry-name`]::[`route-set-name`] or [`registry-name`]::[`as-set-name`] or [`as-number`] or [`registry-name`]::[`route-set-name`][`range-operator`])

{:vspace}
Type:
: optional, multi-valued

## Resolving members through `src-members`

When IRR software is resolving the members of a set which has a `src-members`
attribute, the resolver MUST NOT consider the contents of the `members` or `mp-members`
attribute.

To find the referred set, the resolver MUST match on both the IRR registry
name and the set's primary key.
If the IRR registry is unknown to the resolver, no set can match the reference.

When an as-set/route-set does not contain an `src-members` attribute, the resolver
SHOULD consider the values of `members` and `mp-members`.
If references are found to a set, and there are multiple sets
with this primary key known to the resolver, the behavior is
not defined by this document.


# Relation to `(mp-)members`

Existing IRR software will not be aware of the new `src-members` attribute
and instead refer to `(mp-)members`.
This is also why the existing attributes can not be modified - this software
would consider e.g. `RIPE::AS-EXAMPLE` as the full primary key, and fail
to look up the reference as intended.

Existing IRR objects may also not be updated with `src-members` for some time,
as this can not be done automatically.
Deployment in both software and objects will be a gradual process, however,
even partial deployment will reduce the potential for issues from reference
mixups.

In order to keep support for existing IRR software, the contents of
`src-members` must match `(mp-)members` as close as possible,
which the IRR server can ensure.

## Additional validation

When an authoritative IRR registry processes a set object with a `src-members`
attribute, it MUST ensure that the union of values in `members` and `mp-members`
is equal to the values of `src-members` with the registry names removed from set
references. All values MUST be combined, regardless if they were listed in
one attribute, or in multiple repetitions of the attribute.

This ensures that the new `src-members` can be used, providing the benefits
for updated software, while still having a consistent `(mp-)members` available
for older software.

IRR registry software is RECOMMENDED to make the `src-members` attribute
mandatory on all new as-set/route-set objects, and MAY make it required when modifying
existing objects.

Example of a valid object\:

~~~~ rpsl
route-set: RS-EXAMPLE
members: 192.0.2.0/24
mp-members: 2001:db8::/32
mp-members: RS-OTHER
src-members: 192.0.2.0/24, RIPE::RS-OTHER, 2001:db8::/32
source: EXAMPLE
~~~~
{: title='Valid object'}

Example of an invalid object\:

~~~~ rpsl
route-set: RS-EXAMPLE
members: 192.0.2.0/24
mp-members: 2001:db8::/36
mp-members: RS-OTHER, RS-MPMBRONLY
src-members: 192.0.2.0/24, RIPE::RS-OTHER
src-members: NTTCOM::RS-SRCMBRONLY, 2001:db8::/32
source: EXAMPLE
~~~~
{: title='Invalid object: inconsistent inclusion of RS-MPMBRONLY, NTTCOM::RS-SRCMBRONLY, and 200:db8::/36 vs /32'}

## Automatic generation of `(mp-)members`

Managing multiple copies of the same records is tedious for users.
Therefore, IRR registry software is RECOMMENDED to automatically
fill `(mp-)members`, if not specified by the user, based on `src-members`.

Specifically, for authoritative IRR registries\:

* It is RECOMMENDED that when processing a route-set object
  with a `src-members` attribute, but without both a `members` and `mp-members`
  attribute, the software fills the `mp-members` attribute automatically with the contents
  of `src-members`, with IRR registry prefix removed from references.
* It is RECOMMENDED that when processing an as-set object
  with a `src-members` attribute, but without a `members`
  attribute, the software fills the `members` attribute automatically with the contents
  of `src-members`, with IRR registry prefix removed from references.

The registry MAY return an informational message to the user about
the modifications.
The objects MUST NOT be modified if already submitted with any
`members` or `mp-members` attribute, though the validation rules noted
above must still be applied.

IRR registry software MUST NOT attempt to automatically derive
`src-members` from `(mp-)members`, as this can not be done reliably.

## Multiple references to the same primary key

_Note for review\: this option is questionable. The risk is that the difference_
_in interpretation between legacy members and new src-members increases._


Adding a IRR registry scope to each reference allows a new behavior:

~~~~ rpsl
as-set: AS-EXAMPLE
members: AS-OTHER
src-members: RIPE::AS-OTHER, ARIN::AS-OTHER
source: EXAMPLE
~~~~
{: title='Valid object using multiple registry prefixes with the same RPSL primary key'}

Assuming both RIPE and ARIN have an as-set with primary key `AS-OTHER`, the
{{RFC2622}} interpretation of this `members` attribute is ambiguous in which
object is referenced. It is however clear that it refers to one of them.

The new option of `src-members` not only allows defining which single object
was meant to be referenced, but also allows operators to intentionally include
multiple objects that have the same primary key in their respective IRR registries.


# IANA Considerations {#IANA}

This memo includes no request to IANA.

# Security Considerations {#Security}

This document removes a potential security issue where routing
policy could be manipulated by maliciously creating set objects,
which could be picked over legitimate objects.

While not a new issue, references between set objects may be
circular, and software must detect such cases while resolving.
It may also choose to limit depth or size of their resolving
to avoid excessive resource use.

--- back
