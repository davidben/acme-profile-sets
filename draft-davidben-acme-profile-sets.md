---
title: "Automated Certificate Management Environment (ACME) Profile Sets"
abbrev: "ACME Profile Sets"
category: std

docname: draft-davidben-acme-profile-sets-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Automated Certificate Management Environment"

author:
 -
    fullname: "David Benjamin"
    organization: "Google LLC"
    email: "davidben@google.com"

normative:

informative:

...

--- abstract

This document defines how an ACME Server may indicate collections of related certificate profiles to ACME Clients.

--- middle

# Introduction

As PKIs evolve, an application may require multiple certificate profiles to satisfy the range of relying parties it supports. For example, in a certificate deployment transitioning to post-quantum cryptography, newer relying parties may expect post-quantum trust anchors, while older relying parties still only support classical ones. In other deployments, when a certification authority (CA) is found untrustworthy or its keys rotated or compromised, an application may use a certificate from a newer CA with newer relying parties that have removed the old CA, but still provision a certificate from the older CA with older relying parties that do not trust any newer CAs.

{{!I-D.ietf-acme-profiles}} defines a mechanism for ACME Clients to choose between different certificate profiles from a single ACME Server. However, in applications that require multiple profiles, an ACME Client must know which profiles are needed. This document extends that mechanism with *profile sets*.

A profile set is a set of profiles, defined by the ACME Server and potentially updated over time. An ACME Client configured with a profile set can request a certificate for each profile, transparently updating its certificate set as the profile set evolves.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Extensions to the Directory Resource

An ACME Server that wishes for clients to combine multiple certificate profiles MUST include a new field, `profileSets`, in the `meta` field of its Directory object. The field is defined as follows:

`profileSets` (optional, object):
: A map of profile set names to objects describing profile sets. Profile and profile set names MUST NOT overlap.

Each profile set is described by a JSON object with the following fields:

`description` (required, string):
: A human-readable description of the profile set.

`profiles` (required, array of string):
: An array of strings, containing the names of the profiles in the profile set.

The human-readable descriptions are analogous to those of the `profiles` map defined in {{!I-D.ietf-acme-profiles}}. Their contents are up to the CA; for example, they might be prose descriptions of the properties of the profile, or they might be URLs pointing at a documentation site. ACME Clients SHOULD present these profile set names and descriptions to their operator during initial setup and at appropriate times thereafter.

Profile sets MAY contain overlapping profiles.

Together, the `profiles` and `profileSets` maps indicate the choices available to the ACME Client operator. An individual profile in a profile set MAY appear in the `profiles` map if the ACME Server intends it to be a standalone choice for the ACME Client operator. It MAY also be omitted from the `profiles` map if the ACME Server intends it to be used only in a profile set.

For example, the following Directory object defines two profile sets, `profile1Ext` and `profile2`, alongside standalone `profile1`. An ACME Client might interpret this by offering three choices to the operator, `profile1`, `profile1Ext`, and `profile2`, with their corresponding human-readable descriptions.

The `profile1Ext` profile set consists of `profile1`, which is also a standalone profile, and `profileExt`, which is a profile-set-only profile. The `profile2` profile set consists of three profile-set-only profiles, `profile2a`, `profile2b`, and `profileExt`.

~~~
HTTP/1.1 200 OK
Content-Type: application/json

{
  "newNonce": "https://example.com/acme/new-nonce",
  "newAccount": "https://example.com/acme/new-account",
  "newOrder": "https://example.com/acme/new-order",
  "newAuthz": "https://example.com/acme/new-authz",
  "revokeCert": "https://example.com/acme/revoke-cert",
  "keyChange": "https://example.com/acme/key-change",
  "meta": {
    "termsOfService": "https://example.com/acme/terms/2021-10-05",
    "website": "https://www.example.com/",
    "caaIdentities": ["example.com"],
    "externalAccountRequired": false,
    "profiles": {
      "profile1": "https://example.com/docs/profiles#profile1",
    },
    "profileSets": {
      "profile1Ext": {
        "description":
            "https://example.com/docs/profiles#profile1Ext",
        "profiles": ["profile1", "profileExt"]
      },
      "profile2": {
        "description":
            "https://example.com/docs/profiles#profile2",
        "profiles": ["profile2a", "profile2b", "profileExt"]
      }
    }
  }
}
~~~

# Client Behavior

An ACME Client that supports profile sets MAY be configured to obtain certificates according to a profile set, rather than an individual profile.

If configured with a profile set, the ACME Client SHOULD request certificates from each profile in the profile set, creating independent, parallel orders for each. Each profile MAY spend different amounts of time in the "processing" state, so the ACME Client SHOULD support multiple orders in the "processing" state in parallel. Each profile MAY produce certificates with different lifetimes, so the ACME Client SHOULD evaluate each certificate for renewal independently.

The ACME Client SHOULD periodically re-fetch the Directory object to discover updated profile set definitions. For example, the client MAY re-fetch the Directory when it periodically evaluates its certificates for renewal. If the profile set definition has since changed, the ACME Client SHOULD request certificates for any newly-added profiles. Certificates for newly-removed profiles SHOULD NOT be removed until the ACME Client has provisioned some certificate for each profile in the new profile set definition.

ACME Clients MAY implement the above with the following example procedure, run periodically:

1. Fetch the Directory object from the ACME Server.

2. For each profile in the selected profile set:

   a. Check if the client has an unfinished order for the profile. If so, check if it has entered the "valid" state and download the certificate.

   b. Otherwise, check if the client already has a certificate for the profile, and if it should be renewed, e.g. because it is soon to expire.

   c. If the certificate does not exist, or is soon to expire, start a new order with the profile, as described in {{!I-D.ietf-acme-profiles}}, and complete its authorizations.

3. Cancel any unfinished orders for profiles that are no longer in the profile set.

4. If the client has a certificate for each profile in the profile set, remove certificates for profiles that are no longer in the profile set.

Determining which certificate to use with which relying party is out of scope for this document. TLS {{?RFC8446}} implementations MAY use the procedures defined in {{Sections 4.4.2.2 and 4.4.2.3 of ?RFC8446}}, as well as other TLS extensions, to select certificates.

# Security Considerations

The extensions to the ACME protocol described in this document build upon the Security Considerations and threat model defined in {{Section 10.1 of !RFC8555}}, along with the mechanism defined in {{!I-D.ietf-acme-profiles}}. It does not change the account management or identifier validation flows, so the security considerations are largely unchanged. It also does not change the lifecycle of any individual order, or issuance from the perspective of the server.

Profile sets allow an ACME Server to help ACME Clients configure themselves appropriately during PKI security transitions, such as a change in algorithm, a change in trusted CAs, or CA key rotation. Most PKIs have far fewer ACME Servers than ACME Clients, with ACME Server operators well-connected to relying party requirements. This can help transitions complete more quickly, and thus allow the PKI to realize the security benefits sooner.

# IANA Considerations

## ACME Directory Metadata Fields

IANA will add the following entry to the "ACME Directory Metadata Fields" registry within the "Automated Certificate Management Environment (ACME) Protocol" registry group at <https://www.iana.org/assignments/acme>:

Field Name     | Field Type | Reference
---------------|------------|-----------
profileSets    | object     | This document

--- back

# Acknowledgements
{:numbered="false"}

Thanks to Aaron Gable for feedback on this document, as well as many valuable design discussions.
