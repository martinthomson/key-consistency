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
    ins: A. Davidson
    name: Alex Davidson
    org: Brave Software
    email: alex.davidson92@gmail.com
 -
    ins: M. Finkel
    name: Matthew Finkel
    org: The Tor Project
    email: sysrqb@torproject.org
 -
    ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net
 -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: caw@heapingbits.net

normative:
informative:
  ODOH: I-D.pauly-dprive-oblivious-doh
  PRIVACY-PASS: I-D.ietf-privacypass-protocol
  PRIVACY-PASS-ARCH: I-D.ietf-privacypass-architecture
  OHTTP: I-D.ietf-ohai-ohttp

--- abstract

This document describes the key consistency and correctness requirements of protocols such as
Privacy Pass, Oblivious DoH, and Oblivious HTTP for user privacy. It discusses several mechanisms
and proposals for enabling user privacy in varying threat models. In concludes with discussion
of open problems in this area.

--- middle

# Introduction

Several proposed privacy-enhancing protocols such as Privacy Pass
{{PRIVACY-PASS}}, Oblivious DoH {{ODOH}}, and Oblivious HTTP {{OHTTP}} require
clients to obtain and use a public key for execution. For example, Privacy Pass
public keys are used by clients for validating privately issued tokens for
anonymous session resumption. Oblivious DoH and HTTP both use public keys to
encrypt messages to a particular server.

User privacy in these systems depends on users receiving a key that many, if not
all, other users receive.  If a user were to receive a public key that was
specific to them, or restricted to a small set of users, then use of that public
key could be used to learn targeted information about the user.  Users
also need to receive the correct public key.

In this document, we elaborate on these core requirements, and survey various system designs that might
be used to satisfy them. The purpose of this document is to highlight challenges in building and deploying
solutions to this problem.

## Requirements

{::boilerplate bcp14-tagged}

# Terminology

This document defines the following terms:

Key Consistency and Correctness System (KCCS):
: A mechanism for providing clients with a consistent view of cryptographic key material within a period of time.

Reliant System:
: A system that embeds one or more key consistency and correctness systems.

The KCCS's consistency model is dependent on the implementation and reliant system's threat model.

# Core Requirements {#reqs}

Privacy-focused protocols which rely on widely shared public keys typically
require keys be consistent and correct. Informally, key consistency is the
requirement that all users who communicate with an entity share the same view
of the key associated with that entity; key correctness is that the key's
secret information is controlled by the intended entity and is not known to be
available to an external attacker.

Some protocols depend on large sets of users with consistent keys for privacy
reasons. Specifically, all users with a consistent key represent an anonymity
set wherein each user of the key in that set is indistinguishable from the
rest. An attacker that can actively cause inconsistent views of keys can
therefore compromise user privacy.

An attacker that can cause a user to use an incorrect key will likely compromise
the entire protocol, not just privacy.

Reliant systems must also consider agility when trying to satisfy these requirements. A naive solution to
ensuring consistent and correct keys is to only use a single, fixed key pair for the entirety of
the system. Users can then embed this key into software or elsewhere as needed, without any additional
mechanics or controls to ensure that other users have a different key. However, this solution clearly
is not viable in practice. If the corresponding key is compromised, the system fails. Rotation must
therefore be supported, and in doing so, users need some mechanism to ensure that newly rotated
keys are consistent and correct.

Operationally, servers rotating keys may likely need to accommodate
distributed system state-synchronization issues without sacrificing availability. Some systems and protocols
may choose to prioritize strong consistency over availability, but this document assumes that availability
is preferred to total consistency.

# Consistency and Correctness at Key Acquisition

There are a variety of ways in which reliant systems may build key consistency and correct systems (KCCS),
ranging in operational complexity to ease-of-implementation. In this section, we survey a number of
possible solutions. The viability of each varies depending on the applicable threat model, external
dependencies, and overall reliant system's requirements.

We do not include the fixed public key model from
{{reqs}}, as this is likely not a viable solution for systems and protocols in practice. In all scenarios,
the server corresponding to the desired key is considered malicious.

## Direct Discovery {#server-based}

In this model, users would directly query servers for their corresponding public key, as shown below.

~~~ aasvg
+----------+              +----------+
|          |              |          |
|  Client  +------------->+  Server  |
|          |              |          |
+----------+              +----------+
~~~
{: #fig-disc-direct title="Direct Discovery Example"}

The properties of this solution depend on external mechanisms in place to ensure consistency or
correctness. Absent any such mechanisms, servers can produce unique keys for users without detection.
External mechanisms to ensure consistency here might include, though are not limited to:

- Presenting a signed assertion from a trusted entity that the key is correct.
- Presenting proof that the key is present in some tamper-proof log, similar to Certificate
  Transparency ({{!RFC6962}}) logs.
- User communication or gossip ensuring that all users have a shared view of the key.

The precise external mechanism used here depends largely on the threat model. If there is a trusted
external log for keys, this may be a viable solution.

## Single Proxy Discovery {#proxy-based}

In this model, there exists a proxy that fetches keys from servers on behalf of multiple users, as shown
below.

~~~ aasvg
+----------+
|          |
|  Client  +-----------+
|          |           |
+----------+           |
                       v
+----------+         +----------+       +----------+
|          |         |          |       |          |
|  Client  +-------->+  Proxy   +------>+  Server  |
|          |         |          |       |          |
+----------+         +-+--------+       +----------+
      x                ^
      x                |
+----------+           |
|          |           |
|  Client  +-----------+
|          |
+----------+
~~~
{: #fig-disc-proxy title="Single Proxy Discovery Example"}

If this proxy is trusted, then all users which request a key from this server are assured they have
a consistent view of the server key. However, if this proxy is not trusted, operational risks may arise:

- The proxy can collude with the server to give per-user keys to clients.
- The proxy can give all users a key owned by the proxy, and either collude with the server to use this
  key or retroactively use this key to compromise user privacy when users later make use of the key.

Mitigating these risks may require tamper-proof logs as in {{server-based}}, or via user gossip protocols.

## Multi-Proxy Discovery {#anon-discovery}

In this model, users leverage multiple, non-colluding proxies to fetch keys from servers, as shown below.

~~~ aasvg
                     +----------+
                     |          |
      +------------->+  Proxy   +------------+
      |              |          |            |
      |              +----------+            |
      |                                      v
+-----+----+         +----------+       +----+-----+
|          |         |          |       |          |
|  Client  +-------->+  Proxy   +------>+  Server  |
|          |         |          |       |          |
+-----+----+         +----------+       +----+-----+
      |                    x                 ^
      |                    x                 |
      |              +----------+            |
      |              |          |            |
      +------------->+  Proxy   +------------+
                     |          |
                     +----------+
~~~
{: #fig-disc-multi-proxy title="Multi-Proxy Discovery Example"}

These proxies are ideally spread across multiple vantage points. Examples of proxies include anonymous
systems such as Tor. Tor proxies are general purpose and operate at a lower layer, on arbitrary
communication flows, and therefore they are oblivious to clients fetching keys. A large set of untrusted
proxies that are aware of key fetch requests ({{proxy-based}}) may be used in a similar way. Depending
on how clients fetch such keys from servers, it may become
more difficult for servers to uniquely target individual users with unique keys without detection.
This is especially true as the number of users of these anonymity networks increases. However, beyond
Tor, there does not exist a special-purpose anonymity network for this purpose.

## Database Discovery {#external-db-based}

In this model, servers publish keys in an external database and clients fetch keys from the database, as shown below.

~~~ aasvg
+----------+
|          |
|  Client  +-----------+
|          |           |
+----------+           |
                       v
+----------+         +-+--------+       +----------+
|          |         |          |       |          |
|  Client  +-------->+ Database +<------+  Server  |
|          |         |          |       |          |
+----------+         +-+--------+       +----------+
     x                 ^
     x                 |
+----------+           |
|          |           |
|  Client  +-----------+
|          |
+----------+
~~~
{: #fig-disc-database title="Database Discovery Example"}

The database is expected to have a table that asserts mappings between server names and keys. Examples
of such databases are as follows:

- An append-only, audited table similar to that of Certificate Transparency {{!RFC6962}}. The log is operated and
  audited in such a way that the contents of the log are consistent for all users. Any reliant system
  which depends on this type of KCCS requires the log be audited or users have some other mechanism for
  checking their view of the log state (gossiping). However, this type of system does not ensure proactive
  security against malicious servers unless log participants actively check log proofs. This requirement
  may impede deployment in practice. Experience with Certificate Transparency shows
  that most implementations have chosen not to check SignedCertificateTimestamps before
  using (that is, accepting as valid) a corresponding TLS certificate.

- A consensus-based table whose assertions are created by a coalition of entities that periodically agree on
  the correct binding of server names and key material. In this model the agreement is achieved via a consensus
  protocol, but the specific consensus protocol is dependent on the implementation.

For privacy, users should either download the entire database and query it locally, or remotely query the database
using privacy-preserving queries (e.g., a private information retrieval (PIR) protocol). In the case where the
database is downloaded locally, it
should be considered stale and re-fetched periodically. The frequency of such updates can likely be infrequent
in practice, as frequent key updates or rotations may affect privacy; see {{validity-periods}} for details.
Downloading the entire database works best if there are a small number of entries, as it does not otherwise
impose bandwidth costs on each client that may be impractical.

# Minimum Validity Periods {#validity-periods}

In addition to ensuring that there is one key at any time, or a limited number keys, any system
needs to ensure that a server cannot rotate its keys too often in order to divide clients into
smaller groups based on when keys are acquired. Such considerations are already highlighted within the
Privacy Pass ecosystem, more discussion can be found at {{PRIVACY-PASS-ARCH}}. Setting a minimum validity
period limits the ability of a server to rotate keys, but also limits the rate of key rotation.

# Separate Consistency Verification

The other schemes described here all attempt to directly limit the number of keys that a client
might accept.  However, by changing how keys are used, clients can impose costs on servers that
might discourage key diversity.

Protocols that have distinctly separate processes for acquiring and using keys might benefit from
moving consistency checks to the usage part of the protocol.  Correctness might be guaranteed
through a relatively simple process, such obtaining keys directly from a server.  A separate
correctness check is then applied before keys are used.

## Independent Verification {#civ}

Anonymous queries to verify key consistency can be used prior to use of keys.  A request for the
current key (or limited set of keys) will reveal if the key that was acquired is different than the
original.  If the key that was originally obtained is not included, the client can abort any use of
the key.

It is important that any validation process not carry any information that might tie it to the
original key discovery process or that the system providing verification be trusted.  A proxy (see
{{proxy-based}}) might be sufficient for providing anonymity, though more robust anonymity
protections (see {{anon-discovery}}) could provide stronger guarantees.  Querying a database (see
{{external-db-based}}) might provide independent verification if that database can be trusted not to
provide answers that change based on client identity.

## Key-Based Encryption {#kbe}

Key-based encryption has a client encrypt the information that it sends to a server, such as a token
or signed object generated with the server keys.  This encryption uses a key derived from the key
configuration, specifically not including any form of key identifier along with the encrypted
information.  If key derivation for the encryption uses a pre-image resistant function (like HKDF),
the server can only decrypt the information if it knows the key configuration.  As there is no
information the server can use to identify which key was used, it is forced to perform trial
decryption if it wants to use multiple keys.

These costs are only linear in terms of the number of active keys.  This doesn't prevent the use of
multiple keys, it only makes their use incrementally more expensive.  Trial decryption costs can be
increased by choosing a time- or memory-hard function such as {{?ARGON2=I-D.irtf-cfrg-argon2}} to
generate keys.

Encrypting this way could provide better latency properties than a separate check.

# Future Work

The model in {{anon-discovery}} seems to be the most lightweight and easy-to-deploy mechanism for
ensuring key consistency and correctness. However, it remains unclear if there exists such an
anonymity network that can scale to the widespread adoption of and requirements of protocols like
Privacy Pass, Oblivious DoH, or Oblivious HTTP. Existing infrastructure based on technologies
like Certificate Transparency or Key Transparency may work, but there is currently no general
purpose system for transparency of opaque keys (or other application data).

# Security Considerations {#sec}

This document discusses several models that systems might use to implement public key discovery
while ensuring key consistency and correctness. It does not make any recommendations for such
models as the best model depends on differing operational requirements and threat models.

--- back
