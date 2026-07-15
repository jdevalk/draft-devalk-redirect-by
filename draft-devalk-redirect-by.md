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
  RFC5234:
  RFC9110:

informative:
  RFC6648:
  RFC8126:
  RFC9651:
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
was responsible for it. The field records the same information as the widely
deployed, non-standard X-Redirect-By header, under a name that follows current
guidance on header field naming.

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
Since version 5.1, WordPress core has emitted X-Redirect-By on every redirect
generated through `wp_redirect()`; `wp_safe_redirect()` delegates to that
function. WordPress exposes the value through a filter so that themes and
plugins can identify themselves {{WP-X-REDIRECT-BY}}. TYPO3 emits the field from
its redirect and shortcut-handling middleware {{TYPO3-REDIRECTS}}. This
demonstrates deployment across multiple independent implementations and a large
installed base; this document does not attempt to quantify the proportion of
all HTTP redirects that carry the field. The header was originally proposed in
{{YOAST-PROPOSAL}}.

{{RFC6648}} discourages the "X-" prefix for newly defined header fields but
makes no recommendation about whether an existing "X-" field ought to migrate.
This document defines Redirect-By as the preferred spelling and registers the
deployed X-Redirect-By spelling so the registry reflects what is on the wire
({{legacy}}, {{iana}}). Existing implementations can add Redirect-By alongside
X-Redirect-By without changing the field value syntax.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The term "redirect" refers to a response with a 3xx (Redirection) status code,
other than 304 (Not Modified), that contains a Location field, as defined in
{{Section 15.4 of RFC9110}}.

The "redirect decision-maker" is the component whose decision determines the
status code and Location field of a redirect response. It is not necessarily the
HTTP server or intermediary that serializes or forwards that response.

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

The field value is a non-empty, human-readable identifier. Its syntax uses the
ABNF notation and core rules of {{RFC5234}}:

~~~ abnf
Redirect-By = redirect-agent
redirect-agent = VCHAR [ *( SP / VCHAR ) VCHAR ]
~~~

That is, the value is a sequence of visible ASCII characters that MAY contain
interior spaces (for example, "Yoast SEO Premium"), with no leading or trailing
space. Redirect-By is a singleton field, not a list. A sender MUST NOT generate
more than one Redirect-By field line in a response. A recipient that receives
more than one Redirect-By field line MUST ignore the field. A comma is an
ordinary character within a redirect-agent value, not a value separator;
recipients MUST NOT split the value on commas.

Deployed implementations send the value as free-form text rather than as a
Structured Field ({{RFC9651}}); see {{structured}} for the rationale.

## Multiple Redirects and Intermediaries

Where a request passes through several components that each redirect, each
redirect is a separate response with its own decision-maker. Because a redirect
is resolved one hop at a time, the value on any single response identifies the
one component that determined the status code and Location field for that hop.

An intermediary that forwards a redirect response without changing its status
code or Location field SHOULD NOT add or modify Redirect-By. An intermediary
that changes either value and thereby becomes the redirect decision-maker MUST
replace any existing Redirect-By value with its own identifier or remove the
field.

# Relationship to X-Redirect-By {#legacy}

The field defined here is deployed at scale under the name X-Redirect-By.
{{RFC6648}} recommends against the "X-" prefix for new fields and against
assuming any semantic distinction based only on whether a name has that prefix.
This document defines the following migration behavior:

- New implementations SHOULD send only Redirect-By.
- Existing implementations that send X-Redirect-By SHOULD add Redirect-By and
  MAY retain X-Redirect-By for compatibility with existing recipients.
- When an implementation sends both names, it MUST send the same redirect-agent
  value in each.
- Recipients SHOULD recognize both names. Where both appear, a valid Redirect-By
  value takes precedence over X-Redirect-By.

X-Redirect-By remains in wide use, but its spelling is legacy and is discouraged
for new implementations. Redirect-By is the preferred spelling going forward.

# Design Considerations

## Not a Structured Field {#structured}

{{RFC9205}} recommends that new HTTP fields use Structured Fields {{RFC9651}}.
Redirect-By intentionally retains the deployed X-Redirect-By value syntax. This
allows existing generators to emit the same value under both names without
transformation and allows recipients to migrate between names without adopting
two value parsers. Structured Field serialization would require quoting and
escaping values that the installed base transmits as unquoted text, causing the
two spellings to differ on the wire. Because the field contains one opaque
identifier and defines no list members or parameters, this document considers
wire-format compatibility more valuable than adopting Structured Fields.

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
  compatibility. Its use by new implementations is discouraged.

--- back

# Acknowledgements
{:numbered="false"}

The convention originated during the migration of *The Guardian*'s website from
guardian.co.uk to theguardian.com, in which multiple systems issued redirects
simultaneously, and was first proposed publicly in {{YOAST-PROPOSAL}}. The author thanks the WordPress core contributors who
implemented X-Redirect-By {{WP-X-REDIRECT-BY}}, and the maintainers of the other
systems that adopted it, for establishing the deployed practice this document
standardizes.
