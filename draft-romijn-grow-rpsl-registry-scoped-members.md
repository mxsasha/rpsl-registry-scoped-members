---
stand_alone: true
ipr: trust200902
cat: info
submissiontype: IETF
area: 
wg:

docname: draft-romijn-grow-rpsl-registry-scoped-members-01
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
- ins: J. Bensley
  name: James Bensley
  org: Inter.link GmbH
  street: Boxhagener Str. 80
  code: 10245
  city: Berlin
  country: DE
  email: james@inter.link


normative:
  RFC2622:
  RFC2725:
  RFC4012:
informative:

--- abstract

This document updates RFC2622 and RFC4012 by specifying `src-members`,
a new attribute on as-set and route-set
objects in the Routing Policy Specification Language (RPSL).
This attribute allows a specific registry to be defined for each member
in a set, avoiding problematic ambiguity when resolving set members.
A new validation rule allows gradual upgrades and backwards compatibility.

--- middle

# Introduction

The Routing Policy Specification Language (RPSL) {{RFC2622}} defines the
as-set and route-set objects, extended in {{RFC4012}}.
These are among the most common objects in the Internet Routing Registry (IRR) system.
These sets can either reference a direct member of the set
(such as an AS number, IP prefix, etc.), or additional sets which themselves have
their own direct members and/or reference yet more sets, ad infinitum.
In both cases, this referencing uses the `members` and `mp-members`
attributes {{RFC4012}}.
Server and client software can follow these references to 
resolve a set down to its members, a set of prefixes or ASes.

A set may refer to another set by including the primary key in its
`(mp-)members` attribute.
The referred set may be in the same or in another IRR registry.
It is not possible to specify the IRR registry of
the referred set. This can lead to primary key collisions
when resolving a set\:

1. There are multiple significant IRR registries.
1. Sets often reference objects in registries other than the registry the set itself is stored in.
1. There is no guaranteed uniqueness of object primary keys amongst the different registries.
1. Hence, multiple objects may exist in the IRR system with the same primary key, 
   making references to them ambiguous.
1. Many IRR servers will mirror data from multiple IRR registries,
   meaning that even within a single server, there are usually collisions.

The ambiguity encountered when resolving set members can result in either an
incorrect RPSL object being chosen, because an object with the same primary key
was retrieved from the wrong IRR registry, or the required RPSL object (which does exist)
is not found, because the resolving process didn't try to retrieve the object from
the correct IRR registry.
"Incorrect" and "wrong" in this context meaning\: not as intended or expected by the operator.

Including members from the incorrect RPSL object can result in the computation of
unintentional routing policy information, which is then deployed to network infrastructure.
That could cause route leaking, or worse, aid in route hijacking.
This has been seen multiple times on the public Internet.

If intended policies are not included, because the object was not found,
prefixes that should be accepted, are not, and the prefixes are not reachable.
  
With either case, routing policy information may end up missing and connectivity 
may be disrupted.

There is no current way to prevent such ambiguity during set member resolution,
both for operators who create the legitimate objects and those who try to resolve them.

Two previous enhancements to reduce set name collisions have been standardized.
However, the problem persists\:

- {{RFC2622}} Section 5.1 defines hierarchical set names, such as `AS65000:AS-EXAMPLE`
  which may also have additional authorization requirements for the referred aut-num.
  However, this authorization only works within a single IRR registry, and doesn't
  allow the correct external IRR to be specified, if the object in question is not
  local to the IRR registry storing the referring set.
- {{RFC2725}} Section 9.6 defines external repository (IRR) references.
  This allows for the correct IRR registry to be specified for a set member object
  by using the SOURCE:: notation however, this syntax isn't supported in the members
  field of set objects.

To solve this, this document adds `src-members` to as-set and route-set objects,
using a IRR registry name prefix with a double colon.
For example\: "RIPE::AS-EXAMPLE", to refer specifically to an object "AS-EXAMPLE"
in the IRR registry "RIPE". This format is already widely used informally by operators,
including in platforms such as PeeringDB.
Continued availability of existing `(mp-)members` attributes
together with new validation rules, ensures backwards compatibility.

## Requirements Language

{::boilerplate bcp14-tagged}

# The `src-members` attribute

## as-set class

The new `src-members` attribute on as-set is similar to the `members` attribute
from {{RFC2622}}, except that the AS set name MUST be prefixed with a registry name
and a double colon.

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

The new `src-members` attribute on route-set is similar to the `mp-members` attribute 
from {{RFC4012}}, except that any references to set names MUST 
be prefixed with a registry name and a double colon.

{:vspace}
Attribute:
: `src-members`

{:vspace}
Value:
: list of ([`ipv4-address-prefix-range`] or [`ipv6-address-prefix-range`] or [`registry-name`]::[`route-set-name`] or [`registry-name`]::[`as-set-name`] or [`as-number`] or [`registry-name`]::[`route-set-name`][`range-operator`])

{:vspace}
Type:
: optional, multi-valued

## Resolving members through `src-members` {#resolving}

IRR software that resolves the members of a set MUST have support for
objects with and without `src-members`.
These objects may be encountered if they were created or updated before
adoption of `src-members`, or the objects have not been updated since.

The resolving process is\:

1. The resolver MUST include all members listed in the `src-members` attribute,
   if any.
   To find the referenced sets, the resolver MUST match on both the IRR registry
   name and the set's primary key.
   If the IRR registry is unknown to the resolver, no set can match the reference.
1. The resolver MUST include all members listed in the `members` and `mp-members`
   attributes, when their primary key was not already listed in `src-members`.
   If there are multiple sets with a primary key known to the resolver,
   the behavior is not defined by this document as this was a previously 
   existing problem.

Note that the restriction to a specific IRR registry name is only used
to select the correct IRR registry to retrieve the referred object and its
attributes. During recursive resolving, if that set has references to further sets,
those MUST be retrieved from a potentially different registry. This could be either the
registry specified in the `src-members` attribute, if present, or the existing source
selection algorithm the IRR server currently uses when resolving using `(mp-)members`.
In other words, the restriction of the lookup to a specific IRR registry does not cascade.

### Resolving example

~~~~ rpsl
route-set: RS-FIRST
members: RS-SECOND
mp-members: RS-LEGACY
src-members: RIPE::RS-SECOND
source: EXAMPLE

route-set: RS-SECOND
members: RS-THIRD
source: RIPE

route-set: RS-SECOND
members: AS65002
source: OTHER

route-set: RS-THIRD
members: AS65000
source: OTHER

route-set: RS-LEGACY
members: AS65001
source: OTHER
~~~~
{: title='Example objects for recursive lookups and attribute interactions'}

To perform a recursive lookup of RS-FIRST, the IRR software will\:

1. Look up RS-FIRST.
1. Determine that the members of RS-FIRST are RS-SECOND (RIPE registry only)
   and RS-LEGACY (any registry). The mention of RS-SECOND in the `members`
   attribute is not included, as `RS-SECOND` is already listed in `src-members`.
1. Look up RS-SECOND in RIPE, and RS-LEGACY in any registry.
1. Determine that the members of RS-LEGACY are AS65001, and the members
   of RS-SECOND are RS-THIRD.
1. Look up RS-THIRD in any registry.
1. Determine that the members of RS-THIRD are AS65000.

If all mentioned registries are enabled, RS-FIRST would resolve to
AS65000 and AS65001.

The RS-SECOND object in the OTHER registry is never looked up,
and AS65002 is not included. This would happen even if there was no
RS-SECOND object found in RIPE.

### Query parameters

When querying a set to resolve its members, IRR software typically
expects the primary key of the set as a query parameter from the user.
This reference is also ambiguous without referring to a specific registry.

IRR software MUST support a user-provided parameter to restrict the initial lookup
of the set object to a specific registry.
This restriction MUST NOT apply to further lookups performed by the IRR software
when resolving the set, i.e., this restriction also does not cascade.

For text based queries, IRR software is RECOMMENDED to allow this parameter to be
provided in similar syntax as the `src-members` value, e.g. RIPE::AS-DEMO,
to query the object with primary key AS-DEMO in registry RIPE,
and then continue resolving its members.

# Relation to `(mp-)members`

Existing IRR software will not be aware of the new `src-members` attribute
and instead refer to `(mp-)members`.
This is also why the existing attributes are not modified - this existing software
would consider e.g. `RIPE::AS-EXAMPLE` as the full primary key of a set, and fail
to look up the reference as intended.

Existing IRR objects may also not be updated with `src-members` for some time,
as this cannot be done automatically. Or, they may be partially updated
as for large sets, finding the intended IRR registry references
may take some time.
Deployment in both software and objects will be a gradual process, however,
even partial deployment will reduce the potential for issues from reference
mixups.

In order to keep support for existing IRR software, the contents of
`src-members` must match `(mp-)members` as close as possible,
which the IRR server will ensure.

## Additional validation  {#validation}

When an authoritative IRR registry processes a set object with a `src-members`
attribute, it MUST validate that all references in `src-members`, with the registry
names removed, are also listed in `members` or `mp-members`.
All values MUST be combined, regardless if they were listed in
one attribute, or in multiple repetitions of the attribute.

This ensures that the new `src-members` can be used, providing the benefits
for updated resolver software, while still having a consistent `(mp-)members` available
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
{: title='Invalid object: inconsistent inclusion of NTTCOM::RS-SRCMBRONLY, and 200:db8::/36 vs /32. Note: the inclusion RS-MPMBRONLY only in mp-members is permitted.'}

## Automatic generation of `(mp-)members`

Managing multiple copies of the same records is tedious for users.
Therefore, IRR registry software is RECOMMENDED to automatically
fill `(mp-)members`, if not specified by the user, based on `src-members`,
in authoritative objects, when the user creates or updates the object.

Specifically, for authoritative IRR registries\:

* It is RECOMMENDED that when creating/updating a route-set object
  with a `src-members` attribute, but without both a `members` and `mp-members`
  attribute, the software fills the `mp-members` attribute automatically with the contents
  of `src-members`, with the IRR registry prefix removed from references.
* It is RECOMMENDED that when creating/updating an as-set object
  with a `src-members` attribute, but without a `members`
  attribute, the software fills the `members` attribute automatically with the contents
  of `src-members`, with the IRR registry prefix removed from references.

The registry MAY return an informational message to the user about
the modifications.
The objects MUST NOT be modified if already submitted with any
`members` or `mp-members` attribute, though the [validation rules noted
earlier](#validation) MUST still be applied.
Non-authoritative servers MUST NOT generate `members` or `mp-members`
automatically.

IRR registry software MUST NOT attempt to automatically derive
`src-members` from `(mp-)members`, as this cannot be done reliably.


## Multiple references to the same primary key

Adding a IRR registry scope to each reference syntactically allows a new
behavior\: having multiple references to the same RPSL primary key.
This is not permitted, and IRR registry software MUST reject this\:

~~~~ rpsl
src-members: RIPE::AS-OTHER, ARIN::AS-OTHER
~~~~
{: title='Invalid object fragment using multiple registry prefixes with the same RPSL primary key'}

The IRR registry software MUST verify that, without their registry prefix,
all references from `src-members` are unique.

This reduces ambiguity regarding backwards compatibility with `(mp-)members`
described earlier.
If allowed, the attribute `src-members: RIPE::AS-OTHER, ARIN::AS-OTHER` would
refer to two different sets, whereas the translation `mp-members: AS-OTHER`
only refers to one set.

# IANA Considerations {#IANA}

This memo includes no request to IANA.

# Security Considerations {#Security}

This document removes a potential security issue where routing
policy could be manipulated by maliciously creating set objects,
which could be used in favor of legitimate objects.

While not a new issue, references between set objects can be
circular, and software MUST detect such cases while resolving.
It is RECOMMENDED to also limit the depth or size of their resolving
to prevent excessive resource use.

--- back
