---
title: Key Consistency and Discovery
abbrev: Key Consistency and Discovery
docname: draft-wood-key-consistency-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: caw@heapingbits.net

normative:
  RFC7748:
informative:
  ODOH: I-D.pauly-dprive-oblivious-doh
  OHTTP: I-D.thomson-http-oblivious
  PRIVACY-PASS: I-D.ietf-privacypass-protocol

--- abstract

This document describes the key consistency and correctness requirements of protocols such as
Privacy Pass, Oblivious DoH, and Oblivious HTTP for user privacy. It discusses several mechanisms
and proposals for enabling user privacy in varying threat models. In concludes with discussion
of open problems in this area.

--- middle

# Introduction

Several proposed privacy-enhancing protocols such as Privacy Pass {{PRIVACY-PASS}},
Oblivious DoH {{ODOH}}, and Oblivious HTTP {{OHTTP}}
require clients to obtain and use a public key for execution. For example, Privacy Pass public keys
are used to privately obtain tokens for anonymous resumption. Oblivious DoH and HTTP both use public
keys to encrypt messages to a particular server. In all cases, a common security goal is that recipients
cannot link usage of a public key to a specific (set of) user(s). In other words, all users of a public
key should belong to the same anonymity set, and an attacker should not be able to actively reduce the
size of this anonymity set. Moreover, an attacker should not be able to convince users to use a key that
does not belong to the intended server.

In this document, we elaborate on these core requirements, and survey various system designs that might
be used to satisfy them. The purpose of this document is to highlight challenges in building and deploying
solutions to this problem.

## Requirements

{::boilerplate bcp14}

# Core Requirements {#reqs}

Privacy-focused protocols which rely on widely shared public keys typically require keys be consistent
and correct. Informally, key consistency is the requirement that all users of a key share the
same view of the key. Some protocols depend on large sets of users with consistent keys for privacy
reasons. Specifically, all users with a consistent key represent an anonymity set wherein each user of
the key in that set is indistinguishable from the rest. An attacker that can actively cause inconsistent
views of keys can therefore compromise user privacy.

An attacker may separately cause privacy problems by forcing an users in ananonymity set to use
an incorrect key. Informally, a key is correct if it belongs to the intended server and is not otherwise
available to an attacker. In a public key setting, this means that the public key is that which
is owned by the corresponding owner, and only that owner has access to the private key. An attacker
that can actively cause users to make use of incorrect keys may be able to compromise user privacy.

Systems must also consider agility when trying to satisfy these requirements. A naive solution to
ensuring consistent and correct keys is to only use a single, fixed key pair for the entirety of
the system. Users can then embed this key into software or elsewhere as needed, without any additional
mechanics or controls to ensure that other users have a different key. However, this solution clearly
is not viable in practice. If the corresponding key is compromised, the system fails. Rotation must
therefore be supported, and in doing so, users need some mechanism to ensure that rotated newly rotated
keys are consistent and correct. Operationally, servers rotating keys may likely need to accommodate
distribuetd system state synchronization issues without sacrificing availability. Some systems and protocols
may choose to prioritize strong consistency over availability, but this document assumes that availability
is preferred to consistency.

# Deploying Consistency and Correctness

There are a variety of ways in systems may build systems for ensuring key consistency and correctness,
ranging in operational complexity to ease-of-implementation. In this section, we survey a number of
possible solutions. The viaibility of each varies depending on the applicable threat model, external
dependencies, and overall system requirements. We do not include the fixed public key model from
{{reqs}}, as this is likely not a viable solution for systems and protocls in practice. In all scenarios,
the server corresponding to the desired key is considered malicious.

## User-Driven Discovery {#user-based}

In this model, users would directly query servers for their corresponding public key. The properties
of this solution depend on external mechanisms in place to ensure consistency or correctness. Absent
any such mechanisms, servers can produce unique keys for users without detection. External mechanisms
to ensure consistency here might include, though are not limited to:

- Presenting proof that the key is present in some tamper-proof log, similar to Certificate
  Transparency logs.
- User communication or gossip ensuring that all users have a shared view of the key.

The precise external mechanism used here depends largely on the threat model. If there is a trusted
external log for keys, this may be a viable solution.

## Proxy-Based Key Discovery {#proxy-based}

In this model, there exists a proxy that fetches keys from servers on behalf of multiple users. If this
proxy is trusted, then all users which request a key from this server are assured they have a consistent
view of the server key. However, if this proxy is not trusted, operational risks may arise:

- The proxy can collude with the server to give per-user keys to clients.
- The proxy can give all users a key owned by the proxy, and either collude with the server to use this
  key or retroactively use this key to compromise user privacy when users later make use of the key.

Mitigating these risks may require tamper-proof logs as in {{user-based}}.

## Log-Based Discovery

In this model, servers publish keys in a tamper-proof log similar to that of Certificate Transparency.
Users may then fetch keys directly from the server and subsequently verify their existence in the log.
The log is operated and audited in such a way that the contents of the log are consistent for all users.
Any system which depends on this type of system requires the log be autited or users have some other
mechanism for checking their view of the log state (gossiping). However, this type of system does not
ensure proactive security against malicious servers unless log participants actively check log proofs.
This requirement may impede deployment in practice, given that no web browser checks
SignedCertificateTimestamps before using (accepting as valid) a corresponding TLS certificate.

## Anonymous System Discovery {#anon-discovery}

In this model, users leverage an anonymity network such as Tor to fetch keys directly from servers
over multiple vantage points. Depending on how clients fetch such keys from servers, it may become
more difficult for servers to uniquely target individual users with unique keys without detection.
This is especially true as the number of users of these anonymity networks increases. However, beyond
Tor, there does not exist a special-purpose anonymity network for this purpose.

# Future Work

The model in {{anon-discovery}} seems to be the most lightweight and easy-to-deploy mechanism for
ensuring key consistency and correctness. However, it remains unclear if there exists such an
anonymity network that can scale to the widespread adoption of and requirements of protocols like
Privacy Pass, Oblivious DoH, or Oblivious HTTP. Existing infrastructure based on technologies
like Certificate Transparency or Key Transparency may work, but there is currently no general
purpose system for transparency of opaque keys (or other application data).

# Security Considerations {#sec}

This document discusses several mdoels that systems might use to implement public key discovery
while ensuring key consistency and correctness. It does not make any recommendations for such
models as the best model depends on differing operational requirements and threat models.

--- back
