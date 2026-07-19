---
title: "The Redirect-By HTTP Response Header Field"
abbrev: "Redirect-By"
docname: draft-devalk-redirect-by-latest
category: info
submissiontype: IETF
ipr: trust200902
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
  - redirect
  - debugging
  - observability
  - http header

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  - fullname: Joost de Valk
    initials: J.
    surname: de Valk
    organization: Emilia Capital
    email: joost@emilia.capital
    uri: https://joost.blog/

normative:
  RFC9110:
  RFC9651:

informative:
  RFC6648:
  RFC8126:
  RFC9205:
  WP-X-REDIRECT-BY:
    title: "x_redirect_by filter"
    target: https://developer.wordpress.org/reference/hooks/x_redirect_by/
    author:
      - org: WordPress
  TYPO3-REDIRECTS:
    title: "TYPO3 redirect and shortcut middleware emitting X-Redirect-By"
    target: https://docs.typo3.org/c/typo3/cms-redirects/main/en-us/
    author:
      - org: TYPO3
  YOAST-PROPOSAL:
    title: "Let's introduce the X-Redirect-By header"
    target: https://yoast.com/developer-blog/x-redirect-by-header/
    author:
      - name: Joost de Valk

--- abstract

This document defines the Redirect-By HTTP response header field. It allows the
software component whose decision determined an HTTP redirect to identify
itself, so that an operator diagnosing a redirect can determine which component
was responsible for it. The field succeeds the widely deployed, non-standard
X-Redirect-By header, which this document deprecates.

--- middle

# Introduction

An HTTP request for a given URL is frequently redirected by one of several
independent components before it reaches an origin resource: a Content Delivery
Network (CDN) edge rule, a reverse proxy, a load balancer, a Content Management
System (CMS), or an application plugin may each issue a redirect. When such a
redirect is misconfigured, an operator inspecting the response sees only the
status code and the Location field ({{Section 10.2.2 of RFC9110}}). Neither
identifies which component in the chain produced the redirect. Diagnosis then
proceeds by disabling components one at a time until the responsible one is
found, which is slow and error prone.

This document defines the Redirect-By response header field, whose value names
the software component whose decision determined the redirect. Its presence
turns an opaque redirect into a self-describing one: the responsible component
is named in the response itself.

The convention is widely deployed under the non-standard name X-Redirect-By.
Since version 5.1, WordPress core has, by default, emitted X-Redirect-By on
every redirect generated through `wp_redirect()`; `wp_safe_redirect()`
delegates to that function. WordPress exposes the value through a filter so that themes and
plugins can identify themselves {{WP-X-REDIRECT-BY}}. TYPO3 emits the field from
its redirect and shortcut-handling middleware {{TYPO3-REDIRECTS}}. This
demonstrates deployment across multiple independent implementations and a large
installed base; this document does not attempt to quantify the proportion of
all HTTP redirects that carry the field. The header was originally proposed in
{{YOAST-PROPOSAL}}.

{{RFC6648}} discourages the "X-" prefix for newly defined header fields but
makes no recommendation about whether an existing "X-" field ought to migrate.
This document chooses migration: it defines Redirect-By as the successor
field, specified as a Structured Field {{RFC9651}}, and deprecates
X-Redirect-By while registering it so the registry reflects what is on the
wire ({{legacy}}, {{iana}}). Existing implementations can send Redirect-By
alongside X-Redirect-By during the transition ({{legacy}}).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The term "redirect" refers to a response with a 3xx (Redirection) status code
({{Section 15.4 of RFC9110}}), other than 304 (Not Modified), that contains a
Location field.

The "redirect decision-maker" is the component whose decision determines the
Location field of a redirect response. It is not necessarily the HTTP server
or intermediary that serializes or forwards that response.

# The Redirect-By Field {#field}

The "Redirect-By" response header field allows the redirect decision-maker to
identify itself.

A redirect decision-maker MAY include a Redirect-By field in the response. When
present, its value SHOULD be a short, stable identifier for that component, such
as a product, service, or plugin name.

The field is informational. A recipient MUST NOT treat the presence, absence, or
value of Redirect-By as altering the meaning of the redirect; the Location field
and status code retain their full semantics independent of it. A recipient that
does not understand the field ignores it, as with any unrecognized field
({{Section 5.1 of RFC9110}}).

## Syntax

Redirect-By is an Item Structured Field {{RFC9651}}. Its value MUST be a Token
({{Section 3.3.4 of RFC9651}}) or a String ({{Section 3.3.3 of RFC9651}})
naming the redirect decision-maker; this document defines no parameters. For
example:

~~~
Redirect-By: WordPress
~~~

or, for an identifier that a Token cannot carry:

~~~
Redirect-By: "Yoast SEO Premium"
~~~

Redirect-By is a singleton field, not a list. A sender MUST NOT generate more
than one Redirect-By field line in a response. A recipient MUST process the
field as a Structured Field Item; if that parsing fails, or the value is
neither a Token nor a String, the recipient MUST ignore the field. This rule
also disposes of duplicate field lines: combining them into a single value
({{Section 5.2 of RFC9110}}) produces a value that does not parse as an Item,
so the field is ignored.

The deployed X-Redirect-By field carries the identifier as free-form text
rather than as a Structured Field; see {{legacy}}.

## Multiple Redirects and Intermediaries

Where a request passes through several components that each redirect, each
redirect is a separate response with its own decision-maker. Because a redirect
is resolved one hop at a time, the value on any single response identifies the
one component that determined the Location field for that hop.

An intermediary that forwards a redirect response without changing its Location
field SHOULD NOT add or modify Redirect-By, even if it modifies other parts of
the response, such as the status code. An intermediary that changes the
Location field thereby becomes the redirect decision-maker and MUST replace any
existing Redirect-By value with its own identifier or remove the field.

# Relationship to X-Redirect-By {#legacy}

The information carried by Redirect-By has been deployed at scale under the
non-standard name X-Redirect-By, which carries the identifier as free-form
text: a sequence of visible ASCII characters, possibly with interior spaces,
without Structured Field serialization. {{RFC6648}} recommends against the
"X-" prefix for new fields and against assuming any semantic distinction based
only on whether a name has that prefix. This document deprecates X-Redirect-By
and defines the following migration behavior:

- New implementations SHOULD send Redirect-By and SHOULD NOT send
  X-Redirect-By.
- Existing implementations that send X-Redirect-By SHOULD also send
  Redirect-By, and SHOULD stop sending X-Redirect-By once the recipients they
  serve recognize the new name.
- When an implementation sends both fields, both MUST name the same
  decision-maker, even though the serializations can differ: for example,
  `X-Redirect-By: Yoast SEO Premium` alongside
  `Redirect-By: "Yoast SEO Premium"`.
- Recipients SHOULD recognize both names. A Redirect-By field that is not
  ignored under the rules of {{field}} takes precedence over X-Redirect-By; a
  recipient MAY fall back to X-Redirect-By when Redirect-By is absent or
  ignored.

Deprecation records that use of the X-Redirect-By name is discouraged; it does
not make existing responses that carry it invalid, and recipients are expected
to keep recognizing the legacy name while it remains widely deployed.

# Design Considerations

## A Structured Field {#structured}

{{RFC9205}} recommends that new HTTP fields use Structured Fields {{RFC9651}},
and Redirect-By follows that recommendation. The deployed X-Redirect-By field
predates this document and transmits its value as unquoted free-form text, so
the two fields can differ on the wire: an identifier containing spaces is sent
bare in X-Redirect-By but as a quoted String in Redirect-By. This document
accepts that divergence. Migration already requires senders to adopt a new
field name, so adopting the standard serialization at the same time costs
little, and it buys well-defined parsing, an unambiguous rule for duplicate or
combined field lines ({{field}}), and the ability for future documents to
extend the field with parameters.

# Security Considerations

Redirect-By is a voluntary disclosure of the identity of a redirecting
component. It is not an authentication or integrity mechanism: its value is set
by the sender and can be absent, forged, or copied, so a recipient MUST NOT rely
on it to establish the provenance or trustworthiness of a redirect.

Naming software in a response reveals information about a deployment that can aid
an attacker in fingerprinting it. To limit this exposure, the value SHOULD be a
coarse component identifier and SHOULD NOT include software versions, file
system paths, internal host names, or the specific rule that matched. For
example, "WordPress" is appropriate; "WordPress 6.5.2 via
wp-content/plugins/acme/redirects.php:214" is not. Operators for whom any such
disclosure is unacceptable can omit the field; it is optional.

# Privacy Considerations

The field describes server-side software, not the user. It carries no user
identifier and introduces no new client-side state or tracking surface.

# IANA Considerations {#iana}

IANA is requested to register the following two entries in the "Hypertext
Transfer Protocol (HTTP) Field Name Registry" defined in {{Section 18.4 of
RFC9110}}. Redirect-By is requested as a permanent registration; the
registry's procedure for permanent entries is Specification Required
({{RFC8126}}), for which this document is the specification. X-Redirect-By is
requested with a status of "deprecated", documenting the name under which the
field was widely deployed before this document and recording that its use is
discouraged ({{legacy}}). Registering the ubiquitous legacy name is a record
of existing practice, not the minting of a new "X-" field, and is therefore
consistent with {{RFC6648}}.

Entry 1:

- Field Name: Redirect-By
- Status: permanent
- Structured Type: Item
- Reference: This document
- Comments: Successor to the deprecated X-Redirect-By field.

Entry 2:

- Field Name: X-Redirect-By
- Status: deprecated
- Structured Type: None
- Reference: This document
- Comments: Deprecated legacy name for the Redirect-By field ({{field}}).

--- back

# Acknowledgements
{:numbered="false"}

The convention originated during the migration of *The Guardian*'s website from
guardian.co.uk to theguardian.com, in which multiple systems issued redirects
simultaneously, and was first proposed publicly in {{YOAST-PROPOSAL}}. The author thanks the WordPress core contributors who
implemented X-Redirect-By {{WP-X-REDIRECT-BY}}, and the maintainers of the other
systems that adopted it, for establishing the deployed practice this document
standardizes.
