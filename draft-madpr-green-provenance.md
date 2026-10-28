---
title: "Provenance Traceability for the GREEN Controller YANG Model"
abbrev: "GREEN-Provenance"
category: std
docname: draft-madpr-green-provenance-latest
submissiontype: IETF
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Getting Ready for Energy-Efficient Networking"
kw:
 - GREEN
 - provenance
 - COSE
 - YANG
 - energy
venue:
  group: "Getting Ready for Energy-Efficient Networking"
  type: "Working Group"
  mail: "green@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/green/"
  github: "marisolpalmero/green-provenance"
  latest: "https://marisolpalmero.github.io/green-provenance/draft-madpr-green-provenance.html"

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
 - ins: R. Osterberg
   fullname: Robin Osterberg
   organization: Kodeta
   email: robin.ietf@kodeta.se



normative:
  RFC7950:
  RFC9052:
  RFC8341:
  RFC3688:
  RFC6020:
  RFC8345:
  GreenController:
    title: "A YANG Data Model for GREEN Network Controllers"
    author:
      - ins: J. Lindblad
    seriesinfo:
      Internet-Draft: draft-swepcarl-green-controller-yang-00
  ProvenanceDraft:
    title: "Applying COSE Signatures for YANG Data Provenance"
    author:
      - ins: D. Lopez
      - ins: A. Pastor
      - ins: A.H. Feng
      - ins: A.M. Perez
      - ins: H. Birkholz
    seriesinfo:
      Internet-Draft: draft-ietf-opsawg-yang-provenance-06

informative:
  PowerAndEnergy: I-D.ietf-green-power-and-energy-yang
  RFC8340:
  RFC8342:
  GreenTerminology: I-D.ietf-green-terminology
  GreenFramework: I-D.ietf-green-framework

--- abstract

This document defines a YANG module that augments the GREEN network controller YANG model {{GreenController}} to record the result of provenance verification for each Energy Object.

The augmentation builds on the COSE-based signing mechanism defined in {{ProvenanceDraft}}. For each Energy Object, it records whether the most recent provenance signature was valid, which key was used to sign it, and who is responsible for that key: the device itself, the network controller acting on its behalf, or an external authority such as a grid energy provider.

This allows operators and auditors to check not just that energy data is correct, but where it came from, and how much trust applies to each source.

--- middle

# Introduction

The GREEN controller YANG model {{GreenController}} lets a network controller describe the energy and power setup of a network: which nodes and components exist, how energy and power data is collected for them, and how that data relates to the controller's inventory. Operators and auditors use this information for regulatory reporting, renewable energy attribution and carbon accounting. They need to know not only what the data says, but where it came from and whether it can be trusted.

{{ProvenanceDraft}} defines how to protect YANG data with COSE signatures {{RFC9052}}. A `provenance-signature` can accompany YANG instance data, NETCONF notifications, or YANG-Push updates, so that a receiver can verify integrity and origin. It leaves the meaning of the COSE Key Identifier (`kid`) to the signer and the verifier: it is "to be locally agreed, used and interpreted by the signer and the signature validator" ({{ProvenanceDraft}}, Section 3.1). That document says how to sign and verify. It does not say how the result of verification is kept in a management data model.

The three documents fit together as follows:

- {{ProvenanceDraft}} (OPSAWG) defines how data is signed and verified.

- {{GreenController}} defines what energy data a controller collects, from which nodes and components, and how that collection is described.

- This document defines the verification outcome for an Energy Object, or external data, like source of energy, that the controller will need and who owns the signing key. It does not change how signing or verification works, and it does not change how collection is described.

This document augments the energy node attributes of {{GreenController}} with five read-only leaves, defined once in a grouping and used at two levels, the node and the component. The leaves expose:

1. the Key Identifier (`kid`) associated with the most recently verified/received provenance signature;

2. the entity that provisioned and manages the corresponding signing key;

3. whether verification of the most recently received provenance signature succeeded;

4. the time at which provenance verification most recently succeeded; and

5. the reason for the most recent verification failure, when applicable.

Only the outcome of verification is stored. The `provenance-signature` defined by {{ProvenanceDraft}} is not stored in the datastore. It is carried in the transport or notification that delivers the data, and the controller verifies it on receipt.

The controller model describes what is collected, and this draft say whether the collected data is verified. Together they can support the "Read provenance data" step of {{GreenController}}, where a consumer follows the collection graph, compares it with the inventory, and checks for unreported devices or double counting.


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

## Why Augment the Controller Model

Provenance state describes how much trust applies to a piece of energy-related data. It is metadata about that data, not a separate object. For that reason this document augments the energy node attributes of {{GreenController}}, at the Energy Object levels, instead of defining a parallel tree. Data supplied by an external authority, such as a grid operator, is expected to be represented in the controller as an Energy Object, so that the same leaves apply.

An earlier version of this idea targeted the per-device `energy-objects` tree of {{PowerAndEnergy}}. The signature is verified by the controller, and the controller's own model is where the trust decision belongs. A controller that reads many devices, and also reads legacy devices through other means, has one place to look. An earlier target also did not suit the case of a controller that signs for a device that cannot sign.

The leaves are `config false`, so they are operational state. Under the Network Management Datastore Architecture {{RFC8342}}, they are read from the operational datastore of the controller and are never configured. The controller derives their values from its own verification procedure and updates the operational state as telemetry arrives. This is also why they can sit inside a container that is otherwise configuration: `config false` data nested under configuration is permitted by YANG, and NMDA shows it only in the operational datastore.

Device-side YANG does not need to change. A device that signs its own telemetry only needs to produce the COSE signature as described in {{ProvenanceDraft}}. The leaves in this document live on the controller.

## Why a Grouping Used at Node and Component Level

Some signers cover a whole node, for example a router that signs everything it reports. Others cover one component, for example a power supply measured by an external meter whose key is held by the controller. The same five leaves apply in both cases, so they are defined in one grouping and augmented at both levels. A deployment can fill in whichever level matches how signing is done, and leave the other empty.

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

  augment /nw:networks/nw:network/nw:node
            /pen:energy-node-attributes:
    +--ro provenance-key-id?            string
    +--ro provenance-key-owner?         identityref
    +--ro traceability-verified?        boolean
    +--ro traceability-last-verified?   yang:date-and-time
    +--ro traceability-failure-reason?  string
  augment /nw:networks/nw:network/nw:node
            /pen:energy-node-attributes/pen:component:
    +--ro provenance-key-id?            string
    +--ro provenance-key-owner?         identityref
    +--ro traceability-verified?        boolean
    +--ro traceability-last-verified?   yang:date-and-time
    +--ro traceability-failure-reason?  string
~~~

## YANG Module Definition

~~~ yang
~~~ yang
<CODE BEGINS> file "ietf-green-provenance@2026-10-28.yang"
module ietf-green-provenance {
  yang-version 1.1;
  namespace
    "urn:ietf:params:xml:ns:yang:ietf-green-provenance";
  prefix igp;

  import ietf-network {
    prefix nw;
    reference
      "RFC 8345: A YANG Data Model for Network Topologies";
  }

  import ietf-power-energy-network {
    prefix pen;
    reference
      "I-D.swepcarl-green-controller-yang: A YANG Data Model
       for GREEN Network Controllers";
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
    "This module augments the GREEN controller YANG model
     (ietf-power-energy-network) with provenance traceability
     leaves for each energy node and component, exposing the
     outcome of COSE signature verification performed by the
     controller.

     The key words 'MUST', 'MUST NOT', 'REQUIRED', 'SHALL',
     'SHALL NOT', 'SHOULD', 'SHOULD NOT', 'RECOMMENDED', 'NOT
     RECOMMENDED', 'MAY', and 'OPTIONAL' in this document are to
     be interpreted as described in BCP 14 (RFC 2119) (RFC 8174)
     when, and only when, they appear in all capitals, as shown
     here.

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

  revision 2026-10-28 {
    description
      "Initial revision.";
    reference
      "RFC XXXX: Provenance Traceability Augmentation for the
       GREEN Controller YANG Model";
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

  /* Grouping */

  grouping provenance-state {
    description
      "Provenance traceability state. All leaves are set by the
       controller after verifying a COSE provenance signature
       associated with the node or component, per
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
         signature associated with this node or component.  Absent if
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
         for this node or component passed COSE verification.  A
         controller SHOULD NOT use its data in
         any reporting or control pipeline while this leaf is
         false.  Absent if no verification has been attempted.";
    }

    leaf traceability-last-verified {
      type yang:date-and-time;
      config false;
      description
        "The date and time at which the controller last
         successfully verified a provenance signature for this
         node or component.  Absent if no successful verification has
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

  /* Augmentations */

  augment "/nw:networks/nw:network/nw:node"
        + "/pen:energy-node-attributes" {
    description
      "Adds provenance state at node level.";
    uses provenance-state;
  }

  augment "/nw:networks/nw:network/nw:node"
        + "/pen:energy-node-attributes/pen:component" {
    description
      "Adds provenance state at component level.";
    uses provenance-state;
  }
}
<CODE ENDS>
~~~

# Operational Considerations

A controller implementing this augmentation is expected to perform COSE signature verification, per {{ProvenanceDraft}}, upon receipt of telemetry associated with an Energy Object, and to update the five leaves defined in this document accordingly. This document does not mandate a specific verification trigger interval; implementations MAY verify on every received update, or at a configured polling interval, as appropriate to the deployment.

The leaves are only meaningful where the controller has created `energy-node-attributes` for the Energy Object, as described in {{GreenController}}. Where signing covers a whole node, the controller fills the node-level leaves. Where signing covers one component, it fills the component-level leaves. A controller MAY fill both.

The identifier of the node is chosen by the controller. The identity of the underlying Energy Object is held in the `references` of {{GreenController}}. The `provenance-key-id` is not a substitute for either, and a consumer that wants to relate a signature to a device follows the `references`, then the `kid`.

When `traceability-verified` is `false`, this document RECOMMENDS that a controller exclude the associated node or component's data from any energy accounting, carbon reporting, or automated control decision until a subsequent verification succeeds.

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
