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

informative:
  RFC6648:
  RFC8126:
  RFC8941:
  RFC9205:
  WP-X-REDIRECT-BY:
    title: "x_redirect_by filter"
    target: https://developer.wordpress.org/reference/hooks/x_redirect_by/
    author:
      - org: WordPress
  WP-USAGE:
    title: "Usage statistics and market share of content management systems"
    target: https://w3techs.com/technologies/overview/content_management
    author:
      - org: W3Techs
  TYPO3-REDIRECTS:
    title: "TYPO3 redirect and shortcut middleware emitting X-Redirect-By"
    target: https://github.com/TYPO3/typo3/blob/main/typo3/sysext/redirects/Classes/Http/Middleware/RedirectHandler.php
    author:
      - org: TYPO3
  YOAST-PROPOSAL:
    title: "Let's introduce the X-Redirect-By header"
    target: https://yoast.com/developer-blog/x-redirect-by-header/
    author:
      - name: Joost de Valk

--- abstract

This document defines the Redirect-By HTTP response header field. It allows the
software that generates an HTTP redirect to identify itself, so that an operator
diagnosing a redirect can determine which component produced it. The field
records the same information as the widely deployed, non-standard X-Redirect-By
header, under a name that follows current guidance on header field naming.

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
the software component that generated the redirect. Its presence turns an
opaque redirect into a self-describing one: the responsible component is named
in the response itself.

The convention is already deployed on a large share of the web's redirects,
under the non-standard name X-Redirect-By. WordPress core has emitted
X-Redirect-By on redirects since version 5.1, exposing it through a filter so
that themes and plugins can supply their own value {{WP-X-REDIRECT-BY}}; because
WordPress runs a large fraction of all public websites {{WP-USAGE}}, a
correspondingly large share of the web's redirects already carry the field.
TYPO3 emits it from its redirect and shortcut-handling middleware
{{TYPO3-REDIRECTS}}. The header was originally proposed in {{YOAST-PROPOSAL}} and
has since been adopted by further systems.

Because {{RFC6648}} discourages the "X-" prefix for newly defined header fields,
this document specifies the field under the unprefixed name Redirect-By, and
registers both Redirect-By and the deployed X-Redirect-By name so the registry
reflects what is on the wire ({{legacy}}, {{iana}}). Existing implementations are
expected to add Redirect-By alongside X-Redirect-By, sending both during a
transition period.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The term "redirect" refers to a response with a 3xx (Redirection) status code
as defined in {{Section 15.4 of RFC9110}}.

# The Redirect-By Field {#field}

The "Redirect-By" response header field allows the software component that
generated a redirect to identify itself.

A server or intermediary that generates a redirect MAY include a Redirect-By
field in that response. When present, its value SHOULD be a short, stable
identifier for the software component that produced the redirect, such as a
product, service, or plugin name.

The field is informational. A recipient MUST NOT treat the presence, absence, or
value of Redirect-By as altering the meaning of the redirect; the Location field
and status code retain their full semantics independent of it. A recipient that
does not understand the field ignores it, as with any unrecognized field
({{Section 5.1 of RFC9110}}).

## Syntax

The field value is a non-empty, human-readable identifier. Its syntax is defined
using the ABNF of {{Section 5.6.1 of RFC9110}}:

~~~ abnf
Redirect-By = redirect-agent
redirect-agent = VCHAR *( SP / VCHAR )
~~~

That is, the value is a sequence of visible US-ASCII characters that MAY contain
interior spaces (for example, "Yoast SEO Premium"), with no leading or trailing
space. Redirect-By is a singleton field: a redirect response generates at most
one Redirect-By value, naming the component that produced that particular
response. A recipient that receives multiple Redirect-By values, or a
comma-separated list, SHOULD treat only the first as significant.

Deployed implementations send the value as free-form text rather than as a
Structured Field ({{RFC8941}}); see {{structured}} for the rationale.

## Multiple Redirects and Intermediaries

Where a request passes through several components that each redirect, each
component sets Redirect-By on the response it generates. Because a redirect is
resolved one hop at a time, the value observed on any single response identifies
the one component that produced that hop, which is the component an operator is
looking for. An intermediary that forwards a redirect response it did not
generate SHOULD NOT add or modify the Redirect-By field.

# Relationship to X-Redirect-By {#legacy}

The field defined here is deployed at scale under the name X-Redirect-By.
{{RFC6648}} recommends against the "X-" prefix for new fields and against
assuming any semantic distinction between an "X-"-prefixed name and its
unprefixed counterpart. Accordingly:

- New implementations SHOULD send Redirect-By.
- Recipients SHOULD treat Redirect-By and X-Redirect-By as carrying the same
  information. Where both appear, Redirect-By takes precedence.
- An implementation MAY additionally send X-Redirect-By during a transition
  period for the benefit of tools that recognize only the legacy name.

This document does not deprecate X-Redirect-By, which remains in wide use; it
provides the unprefixed name as the preferred spelling going forward.

# Design Considerations

## Not a Structured Field {#structured}

{{RFC9205}} recommends that new HTTP fields use Structured Fields {{RFC8941}}.
Redirect-By does not, because it standardizes an existing deployment in which
values are transmitted as free-form text without the double-quoting a Structured
Field String requires. Requiring Structured Field syntax would render the large
installed base of X-Redirect-By values non-conformant for no operational gain,
since the value is consumed by humans reading a response rather than parsed by
machines. Implementers who prefer to constrain values MAY restrict them to a
token as defined in {{Section 5.6.2 of RFC9110}}.

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
RFC9110}}. Both are requested as permanent registrations; the registry's
procedure for permanent entries is Specification Required ({{RFC8126}}), for
which this document is the specification.

Redirect-By is the name defined and recommended by this document. X-Redirect-By
is registered to document the name under which the field is already widely
deployed. Registering an already-ubiquitous name is a record of existing
practice, not the minting of a new "X-" field, and is therefore consistent with
{{RFC6648}}.

Entry 1:

- Field Name: Redirect-By
- Status: permanent
- Structured Type: None
- Reference: This document
- Comments: Preferred name. Also deployed under the legacy name X-Redirect-By.

Entry 2:

- Field Name: X-Redirect-By
- Status: permanent
- Structured Type: None
- Reference: This document
- Comments: Legacy name for the Redirect-By field ({{field}}), retained for
  compatibility. New implementations use Redirect-By.

--- back

# Acknowledgements
{:numbered="false"}

The convention originated during the migration of *The Guardian*'s website from
guardian.co.uk to theguardian.com, in which multiple systems issued redirects
simultaneously, and was first proposed publicly in {{YOAST-PROPOSAL}}. The author thanks the WordPress core contributors who
implemented X-Redirect-By {{WP-X-REDIRECT-BY}}, and the maintainers of the other
systems that adopted it, for establishing the deployed practice this document
standardizes.
