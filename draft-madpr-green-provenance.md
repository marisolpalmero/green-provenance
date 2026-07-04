---
title: "Provenance Traceability Augmentation for the GREEN Power and Energy YANG Module"
abbrev: "GREEN-Provenance"
category: std
docname: draft-madpr-green-provenance-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: OPS
workgroup: GREEN
kw:
 - GREEN
 - provenance
 - COSE
 - YANG
 - energy
venue:
  group: "Getting Ready for Energy-Efficient Networking"
  type: ""
  mail: "green@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/green/"
  github: "https://github.com/marisolpalmero/green-provenance.git"
  latest: "https://github.com/marisolpalmero/draft-madpr-green-provenance.html"


author:

 - ins: M. Palmero
   fullname: Marisol Palmero
   organization: Independent
   email: marisol.ietf@gmail.com
 - ins: D. Lopez
   fullname: Diego Lopez
   organization: Telefonica
   email: diego.r.lopez@telefonica.com
 - ins: A. Mendez Perez
   fullname: Ana Mendez Perez
   organization: Telefonica
   email: ana.mendezperez@telefonica.com
 - ins: P. Andersson
   fullname: Per Andersson
   organization: Ionio Systems
   email: per.ietf@ionio.se
 - ins: R. Österberg
   fullname: Robin Österberg
   organization: Kodeta
   email: robin.ietf@kodeta.se



normative:
  RFC7950:
  RFC9052:

informative:
  ProvenanceDraft: I-D.ietf-opsawg-yang-provenance
  PowerAndEnergy: I-D.ietf-green-power-and-energy-yang
  RFC8340:
  GreenTerminology: I-D.ietf-green-terminology
  GreenFramework: I-D.ietf-green-framework
  GreenUseCases: I-D.ietf-green-use-cases

--- abstract

This document defines a YANG module that augments the GREEN Power and Energy YANG Module {{PowerAndEnergy}} to record the result of provenance verification for each Energy Object.

The augmentation builds on the COSE-based signing mechanism defined in {{ProvenanceDraft}}. For each Energy Object, it records whether the most recent provenance signature was valid, which key was used to sign it, and who is responsible for that key — whether the device itself, the network controller acting on its behalf, or an external authority such as a grid energy provider.

This allows operators and auditors to verify not just that energy data is correct, but that it came from where it claims to have come from, and to understand the level of trust that applies to each source.

--- middle

# Introduction

The GREEN Power and Energy YANG Module {{PowerAndEnergy}} defines the `energy-objects` operational tree used by a controller to observe power and energy telemetry reported by Energy Objects.

As energy telemetry is increasingly used for regulatory reporting, renewable energy attribution, carbon accounting, and other cross-domain management functions, consumers of this information need to determine not only its content but also its provenance. In these deployments, the ability to identify the origin of telemetry and to verify its integrity becomes part of normal network operations.

{{ProvenanceDraft}} specifies a general mechanism for protecting YANG data using COSE signatures {{RFC9052}}. A `provenance-signature` can accompany YANG instance data, NETCONF notifications, or YANG-Push updates, allowing a receiver to verify the integrity and origin of the signed information. The document intentionally leaves the semantics of the COSE Key Identifier (`kid`) application-specific, stating that it is "to be locally agreed, used and interpreted by the signer and the signature validator" ({{ProvenanceDraft}}, Section 3.1). While that specification defines how signatures are generated and verified, it does not define how the outcome of verification is represented within a management data model.

This document augments the GREEN Power and Energy YANG Module with five read-only leaves under
`/energy-objects/energy-entry`. These leaves expose:

1. the Key Identifier (`kid`) associated with the most recently verified/received provenance signature; 

2. the entity that provisioned and manages the corresponding signing key;

3. whether verification of the most recently received provenance signature succeeded;

4. the time at which provenance verification most recently succeeded; and

5. the reason for the most recent verification failure, when applicable.

The augmentation exposes only the outcome of provenance verification.

This document defines only the outcome of that verification as operational state, not the signature binary itself. The `provenance-signature` defined by {{ProvenanceDraft}} is not stored in the operational datastore. Instead, the signature is carried in the transport or notification that conveys the telemetry and is verified by the controller when received. The rationale for this design is discussed in {{design-rationale}}.

## Motivating Scenarios

The augmentation supports three deployment scenarios:

- Energy Object signs its own telemetry.

- Controller signs on behalf of the Energy Object.

- External authority signs externally supplied information.

# Terminology

{::boilerplate bcp14-tagged}

This document uses the term "Energy Object" as defined in {{GreenTerminology}}.

This document uses the term "kid" (Key Identifier) as defined in {{RFC9052}}, Section 3.1, and as further qualified in {{ProvenanceDraft}}, Section 3.1.

# Design Rationale {#design-rationale}

## Why Augment Rather Than Define a New Container

This document augments the existing `energy-entry` list in {{PowerAndEnergy}} rather than defining a parallel data structure on a new container. Provenance traceability is metadata about an Energy Object's telemetry, not a distinct object in its own.

Augmentation places the new leaves alongside the energy metrics, which is the structure a controller or auditor will expect when querying a single Energy Object.

## Why the Provenance Signature Itself Is Not Stored

An early YANG Doctors review of {{ProvenanceDraft}} raised a specific interoperability concern: two YANG datastore implementations that are independently compliant with {{RFC7950}} MAY serialize a set of augmented leaves in different element orders, particularly when more than one module augments the same parent. Where signature verification depends on re-canonicalizing a complete YANG-level serialization, this divergence can prevent successful cross-implementation verification, even though the underlying content is semantically identical.

This document avoids that problem entirely. The `provenance-signature` binary defined in {{ProvenanceDraft}} is never written into the `energy-objects` operational tree. It is carried only in the transport-level envelope, as a YANG-Push notification, as a NETCONF RPC reply, or as an instance-data document; in which it was received, and is verified upon receipt by the controller at the point of receipt.

This document records only the verification result, not the signature itself, so the interoperability concern above does not apply.

## Why provenance-key-owner and provenance-key-id are both needed
  
A successful provenance verification identifies the signing key used to produce the signature. It does not identify the entity that manages that key.

The GREEN Framework {{GreenFramework}} distinguishes between device-initiated and controller-initiated, with provenance generation:

- In the device-initiated model, the Energy Object generates and manages its own signing key and produces the provenance signature directly. 

- In the controller-initiated model, the controller generates the provenance signature on behalf of an Energy Object that does not provide provenance information itself.

Both models produce a valid COSE signature that can be verified using the associated `kid`, but they represent different trust relationships.

This document defines the `provenance-key-owner` leaf to expose that trust relationship explicitly. Rather than requiring management applications to infer implementation-specific semantics from the `kid`, the controller exposes the entity that manages the signing key associated with the verified signature.

When `provenance-key-owner` is `key-owner-device`, the verified signature was generated using a key managed by the Energy Object, corresponding to the device-initiated provenance model. When `provenance-key-owner` is `key-owner-controller`, the verified signature was generated using a key managed by the controller on behalf of the Energy Object, corresponding to the controller-initiated provenance model. The `key-owner-external-ca` identity extends this model to cover telemetry or energy-related information whose provenance is established by an authority outside the operator's administrative domain, such as a grid energy provider.

Representing the provenance model explicitly as operational state allows controllers, management applications, and auditors to determine the applicable trust relationship without interpreting implementation-specific `kid` values. This also provides a consistent, machine-readable representation across implementations that may use different conventions for assigning `kid` values.

Additional deployment models MAY define identities derived from `key-owner-type` without requiring changes to the module.

## Relationship to provenance-key-id and the underlying COSE kid

The `provenance-key-id` leaf does not define a new identifier. Rather, it exposes the `kid` value carried in the protected header of the `COSE_Sign1` structure defined in {{ProvenanceDraft}} as operational state.

The `kid` identifies the signing key used to produce a signature; it does not identify the Energy Object itself. Unlike a device identifier such as a UUID, which remains stable across key rotations and device restarts, a `kid` may change whenever the corresponding signing key is replaced. As specified by {{RFC9052}}, the format and semantics of the `kid` are application-defined and it is not required to be globally unique.

In deployments where an Energy Object manages its own signing key, the `kid` is chosen by the Energy Object. One suitable representation is a COSE Key Thumbprint URI as specified in RFC 9679, allowing the controller to verify the `kid` independently using the registered public key associated with the Energy Object.

In deployments where the controller signs telemetry on behalf of an Energy Object, the controller assigns the `kid` according to its own administrative namespace. The controller SHOULD maintain a local mapping between the assigned kid value and the corresponding Energy Object identifier.

# YANG Module

## Tree Diagram

The following tree diagram, using the notation defined in {{RFC8340}}, illustrates the augmentation defined by this document.

~~~
module: ietf-green-provenance

  augment /eo:energy-objects/eo:energy-entry:
    +--ro provenance-key-id?            string
    +--ro provenance-key-owner?         identityref
    +--ro traceability-verified?        boolean
    +--ro traceability-last-verified?   yang:date-and-time
    +--ro traceability-failure-reason?  string
~~~

## YANG Module Definition

~~~ yang
<CODE BEGINS> file "ietf-green-provenance@2026-06-26.yang"
module ietf-green-provenance {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-green-provenance";
  prefix igp;

  import ietf-power-and-energy {
    prefix eo;
    reference
      "I-D.ietf-green-power-and-energy-yang: Power and Energy
       YANG Module";
  }

  import ietf-yang-types {
    prefix yang;
    reference
      "RFC 9911: Common YANG Data Types";
  }

  organization
    "IETF GREEN Working Group";

  contact
    "WG Web:  <https://datatracker.ietf.org/wg/green/>
     WG List: <mailto:green@ietf.org>";

  description
    "This module augments the GREEN Power and Energy YANG module
     (ietf-power-and-energy) with provenance traceability leaves
     for each Energy Object, exposing the outcome of COSE
     signature verification performed by the controller.

     Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Revised BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX
     (https://www.rfc-editor.org/info/rfcXXXX); see the RFC
     itself for full legal notices.";

  revision 2026-06-26 {
    description
      "Initial revision.";
    reference
      "RFC XXXX: Provenance Traceability Augmentation for the
       GREEN Power and Energy YANG Module";
  }

  /* Identities */

  identity key-owner-type {
    description
      "Base identity for the entity that provisioned and manages
       the COSE signing key used to sign data associated with an
       Energy Object.";
  }

  identity key-owner-device {
    base key-owner-type;
    description
      "The Energy Object generated and manages its own key pair
       and self-attests its own telemetry.";
  }

  identity key-owner-controller {
    base key-owner-type;
    description
      "The controller generated and manages the key pair on
       behalf of the Energy Object, typically because the Energy
       Object lacks native signing capability.  The controller
       signs data it collected on the Energy Object's behalf.";
  }

  identity key-owner-external-ca {
    base key-owner-type;
    description
      "The key belongs to a certificate authority or data
       authority outside the operator's administrative domain,
       such as a grid energy provider or carbon intensity
       service.";
  }

  /* Augmentation */

  augment "/eo:energy-objects/eo:energy-entry" {
    description
      "Adds provenance traceability metadata to each Energy
       Object entry.  All leaves are set by the controller after
       verifying a COSE provenance signature associated with this
       Energy Object's telemetry, per
       I-D.ietf-opsawg-yang-provenance.  The provenance-signature
       value itself is not stored here; see Section 3.2 of this
       document.";

    leaf provenance-key-id {
      type string {
        length "1..256";
      }
      config false;
      description
        "The Key Identifier (kid) from the COSE_Sign1 protected
         header of the most recently verified provenance
         signature associated with this Energy Object.  Absent if
         no provenance signature has been received and verified.";
      reference
        "RFC 9052: CBOR Object Signing and Encryption (COSE),
         Section 3.1;
         I-D.ietf-opsawg-yang-provenance, Section 3.1";
    }

    leaf provenance-key-owner {
      type identityref {
        base key-owner-type;
      }
      config false;
      description
        "Identifies the entity that provisioned and manages the
         signing key whose kid is recorded in provenance-key-id.
         Absent if no provenance signature has been received and
         verified.";
    }

    leaf traceability-verified {
      type boolean;
      config false;
      description
        "Whether the most recently received provenance signature
         for this Energy Object passed COSE verification.  A
         controller SHOULD NOT use this Energy Object's data in
         any reporting or control pipeline while this leaf is
         false.  Absent if no verification has been attempted.";
    }

    leaf traceability-last-verified {
      type yang:date-and-time;
      config false;
      description
        "The date and time at which the controller last
         successfully verified a provenance signature for this
         Energy Object.  Absent if no successful verification has
         occurred.";
    }

    leaf traceability-failure-reason {
      type string {
        length "0..512";
      }
      config false;
      description
        "A human-readable explanation of the most recent
         provenance verification failure.  Present only when
         traceability-verified is false.";
    }
  }
}
<CODE ENDS>
~~~

# Operational Considerations

A controller implementing this augmentation is expected to perform COSE signature verification, per {{ProvenanceDraft}}, upon receipt of telemetry associated with an Energy Object, and to update the five leaves defined in this document accordingly. This document does not mandate a specific verification trigger interval; implementations MAY verify on every received update, or at a configured polling interval, as appropriate to the deployment.

When `traceability-verified` is `false`, this document RECOMMENDS that a controller exclude the associated Energy Object's data from any energy accounting, carbon reporting, or automated control decision until a subsequent verification succeeds.

# Security Considerations

Implementations MUST ensure that the `traceability-verified` and related leaves defined in this document cannot be set directly via configuration, and are exclusively derived from an actual COSE verification procedure performed by the controller. As these leaves are `config false`, conformant implementations already enforce this at the data model level. Implementers are nonetheless reminded that the integrity of these leaves depends on the correctness of the verification procedure that populates them. Access control mechanisms such as NACM {{RFC8341}} SHOULD restrict write access to the underlying datastore state.

The `key-owner-external-ca` identity introduces a dependency on a trust anchor for the external authority in question. This document does not define a mechanism for provisioning or rotating such trust anchors; this is left to deployment-specific or future specification.

# IANA Considerations

This document registers one URI in the "ns" subregistry of the IETF
XML Registry {{RFC3688}}:

~~~
URI: urn:ietf:params:xml:ns:yang:ietf-green-provenance
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.
~~~

This document registers one YANG module in the "YANG Module Names"
registry {{RFC6020}}:

~~~
name: ietf-green-provenance
namespace: urn:ietf:params:xml:ns:yang:ietf-green-provenance
prefix: igp
reference: RFC XXXX
~~~

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank the participants of the GREEN Working Group design team and the IETF Hackathon sessions where the underlying COSE provenance mechanism was implemented and tested against the scenarios that motivated this document. The authors would also like to thank Jan Lindblad for his YANG Doctors review of {{ProvenanceDraft}}, which directly informed the design rationale in {{design-rationale}} regarding the placement of the provenance signature outside the augmented datastore tree.
