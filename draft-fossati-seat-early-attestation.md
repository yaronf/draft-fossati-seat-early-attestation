---
title: Using Attestation in Transport Layer Security (TLS) and Datagram Transport Layer Security (DTLS)
abbrev: Attestation in TLS/DTLS
docname: draft-fossati-seat-early-attestation-latest
submissiontype: IETF
category: std

ipr: trust200902
area: Security
workgroup: TLS
keyword: [ attestation, RATS, TLS ]

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
  docmapping: yes

venue:
  group: "SEAT"
  type: "Working Group"
  mail: "seat@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/seat/"
  github: "yaronf/draft-fossati-seat-early-attestation"
  latest: "https://yaronf.github.io/draft-fossati-seat-early-attestation/draft-fossati-seat-early-attestation.html"

author:
 -
       ins: Y. Sheffer
       name: Yaron Sheffer
       organization: Intuit
       email: yaronf.ietf@gmail.com

 -
       ins: I. Mihalcea
       name: Ionut Mihalcea
       organization: Arm Limited
       email: Ionut.Mihalcea@arm.com

 -
       ins: Y. Deshpande
       name: Yogesh Deshpande
       organization: Arm Limited
       email: Yogesh.Deshpande@arm.com

 -
       ins: T. Fossati
       name: Thomas Fossati
       organization: Linaro
       email: thomas.fossati@linaro.org

 -
       ins: T. Reddy
       name: Tirumaleswar Reddy
       organization: Nokia
       email: k.tirumaleswar_reddy@nokia.com

normative:
  I-D.ietf-tls-rfc8446bis: tls13
  I-D.ietf-rats-msg-wrap: cmw

informative:
  RFC6960: ocsp
  RFC9334: rats-arch
  I-D.fossati-tls-attestation: old-draft
  I-D.ietf-rats-eat: rats-eat
  I-D.ietf-rats-daa: rats-daa
  I-D.ietf-oauth-selective-disclosure-jwt: sd-jwt
  I-D.ietf-spice-sd-cwt: sd-cwt
  I-D.ounsworth-rats-privacy-framework: rats-privacy
  I-D.ietf-teep-architecture: teep-arch
  I-D.rosomakho-tls-cert-update: cert-update
  RFC5869: hkdf
  TPM1.2:
    target: https://trustedcomputinggroup.org/resource/tpm-main-specification/
    title: TPM Main Specification Level 2 Version 1.2, Revision 116
    author:
      -
        org: Trusted Computing Group
    date: March 2011
  TPM2.0:
    target: https://trustedcomputinggroup.org/resource/tpm-library-specification/
    title: Trusted Platform Module Library Specification, Family "2.0", Level 00, Revision 01.59
    author:
      -
        org: Trusted Computing Group
    date: November 2019
  TLS-Ext-Registry: IANA.tls-extensiontype-values
  TLS-Param-Registry: IANA.tls-parameters
  iana-media-types: IANA.media-types
  iana-content-formats: IANA.core-parameters/content-formats
  I-D.acme-device-attest:
  FIDO-REQS:
    target: https://fidoalliance.org/specs/fido-security-requirements/
    title: "FIDO Authenticator Security Requirements"
    author:
       -
        ins: B. Peirani
        name: Beatrice Peirani
       -
        ins: J. Verrept
        name: Johan Verrept
    date: November 2021
  RA-TLS:
    target: https://arxiv.org/abs/1801.05863
    title: Integrating Remote Attestation with Transport Layer Security
    author:
       -
        ins: T. Knauth
        name: Thomas Knauth
       -
        ins: M. Steiner
        name: Michael Steiner
       -
        ins: S. Chakrabarti
        name: Somnath Chakrabarti
       -
        ins: L. Lei
        name: Li Lei
       -
        ins: C. Xing
        name: Cedric Xing
       -
        ins: M. Vij
        name: Mona Vij
    date: January 2018
  DICE-Layering:
    target: https://trustedcomputinggroup.org/resource/dice-layering-architecture/
    title: DICE Layering Architecture Version 1.00 Revision 0.19
    author:
      -
        org: Trusted Computing Group
    date: July 2020

--- abstract

The TLS handshake protocol allows authentication of one or both peers using static, long-term credentials.
In some cases, it is also desirable to ensure that the peer runtime environment is in a secure state.
Such an assurance can be achieved using remote attestation which is a process by which an entity produces Evidence about itself that another party can use to appraise whether that entity is found in a secure state.
This document describes a TLS extension that enables the negotiation and binding of the TLS authentication key to a remote attestation session.
This enables an entity capable of producing attestation Evidence, such as a confidential workload running in a Trusted Execution Environment (TEE), or an IoT device that is trying to authenticate itself to a network access point, to present a more comprehensive set of security metrics to its peer.
This extension has been designed to allow the peers to use any attestation technology, in any remote attestation topology, and to use them mutually.

--- middle

#  Introduction

Remote Attestation (RA) {{-rats-arch}} is the process by which an entity produces evidence about itself that another party can use to evaluate the trustworthiness of that entity.
This document describes an extension to the TLS handshake that enables the binding of the TLS connection and its authentication key to a remote attestation session.
This enables an attester, such as a confidential workload running in a Trusted Execution Environment (TEE) {{-teep-arch}}, or an IoT device that is trying to authenticate itself to a network access point, to present a more comprehensive set of security metrics to its peer.
This, in turn, allows for the implementation of authorization policies at the relying parties that are based on stronger security signals.

Given the variety of deployed and emerging attestation technologies (e.g., {{TPM1.2}}, {{TPM2.0}}, {{-rats-eat}}) this extension has been explicitly designed to be agnostic to the attestation formats.
This is achieved by reusing the generic encapsulation defined in {{-cmw}} for transporting Evidence and Attestation Results payloads in the `remoteAttestation` extension.

This specification provides both one-way (server-only) and mutual (client and server) authentication using traditional TLS authentication combined with attestation, and allows the attestation topologies at each peer to be independent of each other.
The proposed design supports both background-check and passport topologies, as described in {{Sections 5.2 and 5.1 of -rats-arch}}.
This is detailed in {{negotiating-protocol}}.

The protocol we propose is implemented completely at the TLS level, resulting in several related advantages:

* Implementation is within a single system component.
* Security does not depend on application-level code, which tends to be less secure than widely shared infrastructure components.
* It is easier to reason about the application's security, since the peers' identities and security postures are known as soon as the handshake completes
and the TLS connection is established.
* Application code does not need to change. At most, some configuration is needed, similar to the current use of certificate trust stores.

This document does not mandate any particular attestation technology.

# Conventions and Terminology {#terminology}

The reader is assumed to be familiar with the vocabulary and concepts defined in
{{Section 4 of -rats-arch}}.

The following terms are used in this document:

{: vspace="0"}

The terms "appraise" and "verify" are used with distinctive semantics throughout the document:
: "Appraise" covers the act of checking the validity of Attestation Results or Evidence, as per {{-rats-arch}}, performed by Relying Parties and Verifiers respectively.
"Verify" covers all other checks performed by the two TLS peers, intended to assess the correctness of the cryptographic and protocol operations of the TLS layer.

TLS Identity Key (TIK):
: A cryptographic key used by one of the peers to authenticate itself during the
TLS handshake. The protocol's security is critically dependent on the provenance, lifetime and
protection properties of the TIK. The TIK MUST be the X.509 certificate's end entity key and is maintained and protected by the TEE.

TIK-C, TIK-S:
: The TIK that identifies the client or the server, respectively.

TIK-C-ID, TIK-S-ID:
: An identifier for TIK-C or respectively, TIK-S. This may be a fingerprint
(cryptographic hash) of the public key, but other implementations are possible.

Attestation binder:
: A cryptographic nonce value provided by the TLS stack to the TEE. It is used for binding attestation Evidence to a specific TLS handshake and for providing freshness.

Two-sided uniqueness:
: The property that each peer independently contributes fresh nonces and key-exchange material to the handshake, so that neither peer alone can determine the transcript hash. The attestation binder derived from that transcript is therefore guaranteed to be unique to the specific connection, even if one of the peers is adversarial.

<!-- -->

{::boilerplate bcp14-tagged}

# Overview

The basic functional goal is to link the authenticated key exchange of TLS with an interleaved remote attestation session in such a way that the key used to sign the handshake can be proven to be residing within the boundaries of an attested TEE.
The requirement is that the attester can provide Evidence containing the security status of both the signing key and the platform that is hosting it.
The associated security goal is to obtain such binding so that no replay, relay or splicing from an adversary is possible.

The protocol's security relies on the verifiable binding between the TLS Identity Key, the
specific TLS session
and the platform state through attestation Evidence or Attestation Results conveyed
in the CMW (Conceptual Message Wrapper) {{-cmw}} payload.

## Authentication vs. Attestation

The protocol combines platform attestation with X.509 certificate authentication.

Attestation when used alone is vulnerable to identity spoofing attacks, in particular when zero-day attacks exist for a class of hardware. (TODO: reference). Therefore it needs to be combined with traditional authentication, which in the case of TLS takes the form of CA-signed certificates.

We RECOMMEND that regular applications use authentication and attestation in tandem, to gain the full security guarantees of an authenticated TLS handshake (for the peer/peers being authenticated) as
well as guarantees of platform integrity.

## Integration into the TLS Handshake

The lightweight integration of attestation into the TLS handshake is designed to have
minimal impact on the existing TLS security properties. The changes consist of:

- Negotiation extension: A new `remoteAttestation` TLS extension is added to ClientHello, EncryptedExtensions, CertificateRequest, and Certificate messages to negotiate the use of attestation, indicate supported attestation formats and Verifiers, and carry the attestation credential.

- Independent key derivation: Binder derivation for attestation (see {{crypto-ops}}) is completely independent of the
  regular TLS key schedule. Attestation processing does not affect the standard TLS key derivation and security properties.

This minimal integration approach provides an intuitive explanation of why the
addition of attestation does not adversely affect TLS security. The attestation
components operate independently, leaving the core TLS handshake protocol and
key derivation mechanisms unmodified. Nevertheless, formal validation of these
security properties is still required.

# Attestation Extension

As typical with new features in TLS, the client indicates support for the new extension in the ClientHello message.
The newly introduced extension allows attestation Evidence or Attestation Results to be exchanged.
Freshness of the exchanged Evidence is guaranteed through an Attestation Binder mechanism (see {{crypto-ops}}) when the Background Check Model is in use.
In the Passport Model, freshness expectations are more relaxed and are governed by the lifetime of the signed Attestation Results.

When the extension is successfully negotiated, attestation Evidence or Attestation Results are conveyed in a `remoteAttestation` extension (see {{remote-attestation-extension-section}}).
The CMW payload in the Attestation extension contains the attestation Evidence or Attestation Results encoded according to {{-cmw}}.

The attestation payload MUST contain assertions relating to the attester's TLS Identity Key (TIK-C for client attester, TIK-S for server attester), which associate the private key with the attestation information.
The TEE's signature over the Evidence within the CMW MUST include an attestation binder derived from the message transcript (see {{crypto-ops}}) and the attester's TLS identity public key, as specified in {{remote-attestation-extension-section}}.

The relying party can obtain and appraise the remote Attestation Results either
directly from the Attestation extension (in the Passport Model), or by relaying
the Evidence from the Attestation extension to the Verifier and receiving the
Attestation Results. Subsequent verification of possession of the attested key in the
CertificateVerify message remains unchanged from baseline TLS.

When using the Passport Model, the remote Attestation Results obtained by the
attester from its trusted Verifier can be cached and used for any number of
subsequent TLS handshakes, as long as the freshness policy requirements are
satisfied.

This protocol supports both monolithic and split implementations. In a monolithic
implementation, the TLS stack is completely embedded within the TEE. In a split
implementation, the TLS stack is located outside the TEE, but any private keys
(and in particular, the TIK) only exist within the TEE. In order to support
both options, only the TIK's identity, its public component and a short generated binder are ever
passed between the Client or Server TLS stack and its Attestation Service.
While the two types of implementations offer identical functionality,
their security properties often differ, see {{sec-guarantees}} for more details.

## Remote Attestation Extension {#remote-attestation-extension-section}

As defined in Section 4.4.2 of {{-tls13}}, the TLS `Certificate` message
contains a `certificate_list`, which is a sequence of `CertificateEntry`
structures.

When attestation is negotiated via the extension defined in this document,
the `remoteAttestation` extension defined in this document MUST appear only in
the first `CertificateEntry` of the `Certificate` message and applies
exclusively to the end-entity certificate.

The extension MUST NOT appear in any other `CertificateEntry`.

If the `remoteAttestation` extension is received in any other position, the
receiver MUST abort the handshake with a fatal `illegal_parameter` alert.

This message carries a CMW (Conceptual Message Wrapper) payload as defined in {{-cmw}}.

The `remoteAttestation` extension structure is defined in {{figure-remote-attestation-extension}}.
As per {{Section 4.2 of -tls13}}, a single extension is used across the entire handshake.
The extension is used in ClientHello, EncryptedExtensions, and CertificateRequest messages for protocol negotiation (see {{negotiating-protocol}}).
The extension is used in Certificate messages for carrying attestation credentials.

~~~~
    enum { CONTENT_FORMAT(0), MEDIA_TYPE(1) } typeEncoding;

    struct {
        typeEncoding type_encoding;
        select (EvidenceType.type_encoding) {
            case CONTENT_FORMAT: uint16 content_format;
            case MEDIA_TYPE: opaque media_type<0..2^16-1>;
        };
    } EvidenceType;

    struct {
        opaque verifier_identity<0..2^16-1>;
    } VerifierIdentityType;

    enum { evidence(0), result(1), (255) } AttestationMechanism;

    struct {
        AttestationMechanism mechanism;
        select (mechanism) {
            case evidence: EvidenceType;
            case result:   VerifierIdentityType;
        } argument;
    } AttestationScheme;

    struct {
        select (Handshake.msg_type) {
            case client_hello:
                AttestationScheme server_attester_schemes<0..2^16-1>;

                AttestationScheme client_attester_schemes<0..2^16-1>;

            case encrypted_extensions:
                AttestationScheme chosen_server_scheme;

            case certificate_request:
                AttestationScheme chosen_client_scheme;

            case certificate:
                opaque cmw_payload<1..2^24-1>;
        };
    } remoteAttestation;
~~~~
{: #figure-remote-attestation-extension title="TLS Extension Structure for Remote Attestation negotiation."}

The `cmw_payload` field contains a CMW structure as defined in {{-cmw}}.
Both JSON and CBOR serializations are allowed in CMW, with the emitter choosing
which serialization to use.

The CMW payload MUST contain attestation Evidence (in Background Check Model) or Attestation Results (in Passport Model) that binds the TLS Identity Key (TIK) to the platform and workload state.
The TEE's signature over the Evidence within the CMW MUST include a binder ensuring that the attestation is associated with this particular TLS connection, as well as the attester's TLS identity public key (TIK-C for client attester, TIK-S for server attester).

This binding ensures that the attested key is the one used in the TLS handshake
and provides freshness guarantees through derivation from both peers' randomness.
See {{crypto-ops}} for details.

# Use of Attestation in the TLS Handshake

For both the Passport Model (described in Section 5.1 of {{RFC9334}}) and
Background Check Model (described in Section 5.2 of {{RFC9334}}) the following
modes of operation are allowed when used with TLS, namely:

- TLS client is the attester,
- TLS server is the attester, and
- TLS client and server mutually attest towards each other.

As noted, each peer's attestation is carried in the `remoteAttestation` extension within
that peer's Certificate message. This section describes how the attestation
is produced, bound to the TLS handshake and verified by the recipient.

## Cryptographic Operations {#crypto-ops}

The cryptographic operations defined in this section bind attestation Evidence
to a specific TLS handshake. This binding prevents replay and relay of attestation
Evidence across different TLS connections, and ensures that attestation Evidence
presented during a handshake corresponds to the authenticated
TLS session in which it is conveyed.

The attestation Evidence or Attestation Results are generated by a TEE and
signed using an attestation key. The signed Evidence includes
inputs originating from different trust domains.

The attestation binder is provided by the TLS stack and serves as a
nonce that ensures freshness and binding to a specific TLS handshake,
as well as binding to the attester's TLS public key.

### Attestation Binder Definition

The attestation binder is computed using primitives
defined in Section 4.4.1 and&nbsp;7.1 of {{-tls13}}.

Both peers derive a single attestation base from the same transcript
checkpoint, `ClientHello...ServerHello`.

~~~

attest_base = HKDF-Expand-Label(0, "attestation base",
                                Hash(ClientHello...ServerHello), Hash.length)

c_attest_binder = HKDF-Expand-Label(attest_base, "attestation",
                                    Hash(TLS_Client_Public_Key), Hash.length)
s_attest_binder = HKDF-Expand-Label(attest_base, "attestation",
                                    Hash(TLS_Server_Public_Key), Hash.length)
~~~

`TLS_Client_Public_Key` and `TLS_Server_Public_Key` denote the DER-encoded
SubjectPublicKeyInfo of the peer's end-entity certificate. `Hash` is the
cipher suite hash function for the handshake ({{Section 7.1 of -tls13}}).

We note that `HKDF-Expand-Label` is used to produce binding values rather than keying material. `HKDF-Extract` is not invoked, as there is no input key material to combine. The "0" parameter denotes a byte string of `Hash.length` zeroes.

`HKDF-Expand-Label` is defined in {{Section 7.1 of -tls13}}, which builds on the HKDF construction {{-hkdf}}; its use here does not modify the TLS protocol or the TLS key schedule.

### Verification

Upon receipt of a `remoteAttestation` extension, the peer MUST compute the attestation binder.

If the peer's Evidence is rejected (binder mismatch, failed Evidence appraisal, or malformed CMW),
the receiver MUST send an `attestation_failed` fatal alert and abort the handshake
(see {{tls-alerts}}).

Depending on the architecture (see also {{stack-tee-interface}}), either the peer verifies
the binding or else it delegates this responsibility to an external Verifier.

* In the former case, the peer MUST compare the computed binder value to the attestation binder
included in the
signed Evidence or signed Attestation Results. If the values do not match, the peer MUST treat the
attestation as invalid.

* In the latter case, the RP MUST convey the binder to the
Verifier. The Verifier MUST appraise that the conveyed binder is identical to the one that was signed
in the Evidence or Attestation Results. If appraisal fails, the receiver MUST treat the
attestation as invalid.

<cref>TODO: define a way to transport the binder to a remote Verifier. Possibly
as a (new) conceptual message (CM) within a collection. This would provide
the Verifier whatever information it cannot compute on its own, while
not forcing the TLS stack to parse the Evidence.</cref>

### Security Properties

Binding attestation Evidence to the TLS handshake transcript hash provides the
following security properties:

* Replay protection: Evidence generated for a previous handshake cannot be
  reused in a later handshake.
* Relay protection: Evidence obtained from one TLS connection cannot be
  successfully presented in a different TLS connection, even in the presence of
  a MiTM attacker.

In typical deployments where the TLS handshake executes outside the TEE, a
compromised host can execute the TLS handshake in the rich operating system and
use the TEE as a signing oracle by presenting the attestation binder value to
obtain valid-looking attestation Evidence.

However an endorsed TEE (one that is operating as required by this protocol)
is required to verify the binder against the TLS public key associated
with the private key that it holds. This verification, in conjunction with the TEE's
endorsement being appraised, ensures that relay attacks are prevented.

The attestation binder prevents replay of Evidence across TLS connections. The binding to the TLS identity key ensures that Evidence produced by one endpoint cannot be replayed in a TLS connection involving a different endpoint, as the verifier checks that the binder matches the public key presented in the current TLS connection. The additional binding to the transcript through ClientHello and ServerHello ensures that Evidence cannot be replayed across TLS connections, as ClientHello.random and ServerHello.random are independently generated by each peer for every TLS connection.

## Binding the TIK to the TEE {#tik-binding}

This specification assumes that the TIK private key corresponding to the end-entity certificate used in the TLS handshake is generated inside a TEE and never leaves it. A platform could instead generate the TIK private key outside the TEE and compute the CertificateVerify signature using that external key. A relying party cannot detect this attack unless additional safeguards are in place.

This risk is particularly relevant in split deployments, where the TLS stack does not reside inside the TEE. In such architectures, attesting the TEE alone does not prove that the TIK private key used by the TLS endpoint was generated, is stored, or is controlled by the TEE.

To address this, the signed Evidence MUST include an Attestation Binder generated using the hash of the TIK public key (TIK_pub_hash) (see {{crypto-ops}}).

The Relying Party MUST compute the hash of the TIK public key extracted from the TLS end-entity certificate using
the same hash algorithm and verify that it matches the TIK_pub_hash included in the Evidence. Successful
verification binds the attestation Evidence to the TLS identity used for authentication. This verification is performed by the Relying Party, as the Verifier may not be co-located with the Relying Party and may not have access to the TLS handshake or the TLS end-entity certificate, consistent with the RATS architecture.
Alternatively, in deployments where the Verifier is not co-located with the Relying Party, the Relying Party MAY
supply the Verifier with the hash of the TIK public key. The Verifier then compares this value with the TIK
public key hash included in the Evidence. If the values do not match, the attestation MUST be considered invalid.

Without this binding, a non-TEE TLS endpoint can obtain Evidence from a separate TLS endpoint that genuinely runs
inside a TEE and relay that Evidence to the relying party while executing the TLS handshake itself. If the
Evidence only attests that a TLS stack is running in a TEE, the relying party cannot determine whether the
attested TLS stack is the one that actually performed the handshake. Binding the Evidence to the TIK public key
prevents this relay attack.

The proposed binding ensures that the relying party does not establish a TLS session with a TLS endpoint whose TIK is not generated and controlled by the TEE. It does not - in and of itself - ensure security of the TLS stack when the stack is
outside the TEE, and see {{sec-guarantees}} for a further discussion.

## The TLS Stack's Interface to the TEE {#stack-tee-interface}

When the TEE signs the Evidence or Attestation Results, it also binds them to the TLS Identity public key and the TLS
session. TEE implementations differ, and some only allow a single user-provided challenge value to be added to the Evidence with no associated checks.

Architecturally we propose to add a thin shim between the traditional TLS stack and the TEE
as shown in {{figure-tls-tee-interface}}. Implementations will choose whether to incorporate
the shim into the TEE (making for a "smarter" TEE and better protection
for the remote attestation protocol), or in case of a legacy TEE that cannot be modified,
the shim can be added to the TLS stack.

~~~ aasvg
+----------------------------------------------------+  ------+
|                                                    |        |
|                     TLS Stack                      |        |
|                                                    |        |
+------+---------------------------------------------+        |
       |                         ^                            |
       | Transcript hash         | CMW (Signed                |
       |                         |      Evidence/AR;          |
       | TIK public key hash     |      Nonce)                |
       v                         |                            |
+--------------------------------+-------------------+        |
|                                                    |   Measured &
|              Early Attestation Shim                |    Reported
|                                                    |   Components
+------+---------------------------------------------+        |
       |                         ^                            |
       | Nonce                   | Signed Evidence/AR         |
       v                         |                            |
+--------------------------------+-------------------+        |
|                                                    |        |
|                        TEE                         |        |
| +-----------------+                                |        |
| | TIK Private Key |                                |        |
| +-----------------+                                |        |
+----------------------------------------------------+  ------+
~~~
{: #figure-tls-tee-interface title="TLS Stack Interface with the TEE"}

We adopt a defense-in-depth approach:

* Separate attesting applications within the same TEE SHOULD NOT be capable of impersonating each other via Evidence or Attestation Results. Therefore, if multiple applications are expected to use attestation credentials, evidence/AR generation APIs SHOULD reflect identifiers for the calling contexts into the generated credential. These identifiers can be reflected as separate claims in the credential, or can be measured as part of more generic claims. A Relying Party SHOULD be capable of differentiating between the attesting applications based on their credentials.
* The RP SHOULD NOT base its trust decision only on the Attester's trust root. It SHOULD also ensure that the entire attested software stack is endorsed.
* The TEE itself, when possible, SHOULD generate the attestation secret by running the derivation operations defined in {{crypto-ops}}, and, if it holds the TIK, SHOULD validate the public key. The attestation secret can be generated by the TEE only if TLS is running inside the TEE.
* As shown in the diagram, the TEE itself as well as the TLS stack and the shim SHOULD
all be measured and reported as part of the platform's remote attestation.

## Reattestation {#reattestation}

Attestation Evidence or Attestation Results may become stale over time. For long-lived TLS connections, a relying party may require updated assurance that the peer continues to operate in a trustworthy state.

### Post-Handshake Reattestation Using Client Authentication

Post-handshake client authentication defined in {{Section 4.6.2 of -tls13}} can
be used to obtain updated attestation Evidence or Attestation Results from the TLS client. In this case, the TLS server sends a `CertificateRequest` message after the TLS handshake authentication. The client responds with the standard TLS authentication messages (`Certificate`, `CertificateVerify`, and `Finished`). If attestation has been negotiated for the TLS connection, the client includes the `remoteAttestation` extension in the `Certificate` message carrying updated Evidence or Attestation Results.

The attestation binder can be derived from the post-handshake authentication
transcript defined in Section 4.4 of {{-tls13}}.

This mechanism allows a server to request updated attestation from the client. However, TLS currently does not define a mechanism for post-handshake server authentication. To address this limitation, the subsequent sections discuss design options for handling attestation freshness.

### Option 1: Carrying Attestation in Extended Key Update

One possible approach is to extend the Extended Key Update (EKU) mechanism by introducing a new `ExtendedKeyUpdate` message subtype to carry attestation Evidence or Attestation Results.

However, this approach tightly couples attestation to EKU, even though the two serve different purposes.

### Option 2: No Reattestation (Reconnect for Freshness)

Another approach is to not support reattestation within an established TLS connection. When fresh attestation is required, the client and server terminate the existing TLS session and establish a new one, during which fresh Evidence or Attestation Results are exchanged as part of the handshake.

This approach keeps the TLS protocol unchanged and avoids introducing post-handshake mechanisms. However, it will be disruptive for long-lived TLS connections.

### Option 2: No Reattestation (Make-Before-Break)

Another approach is to not support reattestation within an established TLS connection. When fresh attestation is required, the client establishes a new TLS connection in parallel, exchanging fresh Evidence or Attestation Results as part of the handshake, and migrates application traffic to it before tearing down the old connection.

When the server is the attester and its Evidence needs refreshing, the client signals the server and the client initiates the new connection. When the client is the attester and the server requires fresh Evidence from it, the server signals the client, which then initiates the new connection.

This approach keeps the TLS protocol unchanged and avoids introducing post-handshake mechanisms. Because the new connection is ready before the old one closes, there is no interval without a working connection, unlike tearing down the old connection before establishing the new one, at the cost of briefly maintaining two connections and application-level logic to coordinate the migration.

Note: Since it requires no changes to TLS, it can serve as a workaround while the WG determines which of the other options can be progressed.

### Option 3: Post-Handshake Reattestation Using CertificateUpdate

In this design, reattestation is supported using the `CertificateUpdate` message defined in {{-cert-update}}. Under this approach, the attester sends a `CertificateUpdate` message carrying a new `Certificate` message with updated attestation information. The refreshed attestation is bound to the existing TLS session using post-handshake TLS context.

# Negotiating This Protocol {#negotiating-protocol}

This section defines the TLS extension used to negotiate the use of attestation in the TLS handshake.
Both remote attestation topologies are supported: the Background Check Model, where Evidence is exchanged and appraised during the handshake, and the Passport Model, where pre-appraised Evidence in the form of Attestation Results are presented.
The extension defined in {{figure-remote-attestation-extension}} allows peers to indicate their support for attestation and negotiate which attestation format and, if required, which Verifier to use.

The `remoteAttestation` extension structure contains indicators for both remote attestation topologies, and allows both peers to act as attesters independently during the handshake.

The client selects the remote attestation schemes it supports for both server- or client-as-attester.
The client MUST populate at least one AttestationScheme structure.

The server replies with its preferred schemes for both server- and client-as-attester.
The selected server-as-attester scheme is sent in the EncryptedExtensions message.
While for Background Check the server scheme can be extracted from the CMW sent by the server as part of its Certificate message, for Passport model the Verifier which issued the Attestation Results must be confirmed by the server explicitly.
In order to preserve symmetry and aid the client in handling the server's attestation token, the server explicitly sends its scheme as part of EncryptedExtensions.
The selected client-as-attester scheme is sent in the CertificateRequest message.
The server MUST omit the `remoteAttestation` extension from EncryptedExtensions and CertificateRequest messages if it does not support the corresponding proposed schemes, or if it does not want the corresponding peer to engage in remote attestation.

The `remoteAttestation` extension used to negotiate support for the protocol described in this document is defined in {{figure-remote-attestation-extension}}.

Values for media_type are defined in {{iana-media-types}}.
Values for content_format are defined in {{iana-content-formats}}.
The verifier_identity field can be used to carry an identifier for a Verifier instance.
The identifier needs to be stable across the lifetime of the connection (potentially across Verifier credential rotation), for example a subjectAltName.

# TLS Client and Server Handshake Behavior {#behavior}

The high-level message exchange in {{figure-overview}} shows the `remoteAttestation` extension added to the ClientHello, the EncryptedExtensions, the CertificateRequest, and the Certificate messages.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + key_share*
     | + signature_algorithms*
     | + psk_key_exchange_modes*
     | + pre_shared_key*
     v + remoteAttestation*
     -------->
                                                  ServerHello ^ Key
                                                 + key_share* | Exch
                                            + pre_shared_key* v
                                        {EncryptedExtensions} ^ Server
                                         + remoteAttestation* | Params
                                        {CertificateRequest*} |
                                         + remoteAttestation* v
                                               {Certificate*} ^
                                        + remoteAttestation*  |
                                         {CertificateVerify*} | Auth
                                                   {Finished} v
                               <--------  [Application Data*]
     ^ {Certificate*}
     | + remoteAttestation*
Auth | {CertificateVerify*}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-overview title="Early Attestation Handshake Overview"}

## Client Hello

The `remoteAttestation` extension defined in {{negotiating-protocol}} enables the two peers to use either the Background Check Model or the Passport Model for remote attestation.

To indicate support for either Evidence (for Background Check) or Attestation Results (for Passport), the client includes schemes with either `evidence` or `result` as the AttestationMechanism in the ClientHello extension.
For Evidence, the scheme indicates the expected Evidence type.
For Attestation Results, the scheme indicates the identity of the Verifier from which results can be relayed.
In both cases, whether the scheme is sent as `server_attester_schemes` or `client_attester_schemes` indicates which peer is expected to produce the attestation credential.

The `remoteAttestation` extension carries a list of supported schemes, sorted by preference.
If the client only supports one attestation credential type, it is a list containing a single element.

The client MUST omit schemes from the `client_attester_schemes` field in the extension if it cannot respond to a request from the server to present an attestation credential of the proposed type, or if the client is not configured to use the proposed scheme with the given server.
If the client chooses to include `client_attester_schemes`, it MUST be capable of authenticating itself with a certificate.

For the Background Check Model, the client MUST omit Evidence types from the `server_attester_schemes` field in the extension if it is not able to pass the Evidence type to a Verifier.

## Server Hello

If the server receives a ClientHello that contains the `remoteAttestation` extension, then three outcomes are possible:

-  The server does not support the extension defined in this document.
   In this case, the server returns the EncryptedExtensions without the `remoteAttestation` extension.

-  The server supports the extension defined in this document, but it does not have any remote attestation scheme in common with the client.
   Then, the server terminates the session with a fatal alert of type "unsupported_attestation_schemes".

-  The server supports the extension defined in this document and has at least one remote attestation scheme in common with the client.
   In this case, the processing rules described below are followed.

The `remoteAttestation` extension in the ClientHello indicates the attestation schemes for both peers to act as relying parties.
For schemes conveyed under `server_attester_schemes` the server is expected to act as an attester, while the client is the relying party.
For schemes conveyed under `client_attester_schemes` the server is expected to act as a relying party, while the client is the attester.

If the server chooses to attest itself, it MUST select one of the schemes provided by the client in `server_attester_schemes`.
The server MUST then also include the `remoteAttestation` extension in the EncryptedExtensions message, and MUST include the chosen attestation scheme in the `chosen_server_scheme`.
The server MUST populate the Certificate message extension according to its chosen scheme.
If the server has chosen an `evidence` scheme, the signed Evidence contained in the CMW payload MUST include an Attestation Binder as a nonce value (see {{crypto-ops}}) in the TEE's signature.

Both schemes selected for `chosen_server_scheme` and `chosen_client_scheme` MUST be selected from the schemes provided in the `remoteAttestation` extension sent in the ClientHello.

If both `server_attester_schemes` and `client_attester_schemes` are empty, or if the server does not want to proceed with remote attestation, the server MUST terminate the session as described above, with a fatal alert of type "unsupported_attestation_schemes".

## Certificate Request

If the server chooses to request that the client attests itself, it MUST select one of the schemes provided by the client in `client_attester_schemes`.
The server MUST then also send a CertificateRequest message that includes the `remoteAttestation` extension (see {{figure-remote-attestation-extension}}), and MUST include the chosen attestation scheme in `chosen_client_scheme`.

## Following Server Hello

Upon receipt of the EncryptedExtensions and potentially of the CertificateRequest messages, the client can verify that the server's choices are valid.
The client MUST check that at least one remote attestation scheme was returned, and that the returned schemes were among the corresponding proposed lists.
If the server has rejected that one peer act as an attester by not selecting a corresponding scheme, and the client's policy demands that the remote attestation take place, the client MUST terminate the session with a fatal alert of type "attestation_required".

If the server has selected a valid `chosen_client_scheme`, the client MUST populate the Certificate message extension according to that scheme.
If the server has chosen an `evidence` scheme for the client, the signed Evidence contained in the CMW payload MUST include an Attestation Binder as a nonce value (see {{crypto-ops}}) in the TEE's signature.

# Security Considerations {#sec-cons}

## Relay Resistance {#relay-resistance}

A relay attack succeeds when Evidence produced in one TLS connection can be
presented by a different party in another TLS connection. Preventing it
requires that Evidence be bound to a value that is unique to the TLS connection
in which it is conveyed, so that Evidence generated in any one connection
cannot be replayed in any other.

This mechanism meets that requirement by binding Evidence to the handshake
transcript checkpoint `ClientHello...ServerHello` (see {{crypto-ops}}), a value
that is unique to each connection. The binding does not rely on the application
traffic secrets, nor on any value exported from the completed handshake such as
the Exported Keying Material (EKM; {{Section 7.5 of -tls13}}). EKM is an equally
valid per-connection anchor; the remainder of this section shows that binding to
the transcript checkpoint and binding to EKM provide equivalent relay
resistance.

The two approaches differ only in which connection-unique value the Evidence is
anchored to:

* A post-handshake binding anchors Evidence to EKM, a value that becomes
  available only after the handshake completes.
* The mechanism in this document anchors Evidence to
  `Transcript-Hash(ClientHello...ServerHello)`.

Both anchors are unique per connection. The transcript checkpoint commits to
both peers' fresh ephemeral key-exchange contributions, that is, the client's
and server's `key_share` entries (whether (EC)DHE public keys or a PQC KEM
public key and ciphertext), as well as `ClientHello.random` and
`ServerHello.random`.

Because each peer independently contributes fresh material, neither peer alone
controls the transcript, and the resulting binder is unique to the specific
connection (two-sided uniqueness).

The two constructions therefore offer equal relay resistance.

The relay resistance of this mechanism does not depend on the transcript being secret.
It relies instead on the attestation binder, which is unique to the connection and
bound to the attester's TLS identity key, being carried in Evidence signed by the
TEE (see {{crypto-ops}} and {{tik-binding}}); the confidentiality of ClientHello
and ServerHello is not required.

Although `ClientHello` and `ServerHello` are visible to an on-path observer, an
eavesdropper cannot reuse them to complete a new TLS connection, as it does not know
the ephemeral private keys behind the key shares they carry. Nor can the client or
server reproduce an earlier transcript: each contributes fresh key-exchange material
to every handshake, so every connection yields a different transcript, and therefore a
different binder.

The rationale for anchoring to the transcript rather than to an exporter value is
given in {{transcript-vs-exporter}}.

## Security Guarantees {#sec-guarantees}

We note that as a pure cryptographic protocol, attested TLS as-is only guarantees that the Identity Key is known by the TEE. A number of additional guarantees must be provided by the platform and/or the TLS stack,
and the overall security level depends on their existence and quality of assurance:

* The Identity Key is generated by the TEE.
* The Identity Key is never exported or leaked outside the TEE.
* The TLS protocol, whether implemented by the TEE or outside the TEE, is implemented correctly and (for example) does not leak any session key material.

These properties may be explicitly promised ("attested") by the platform, or they can be assured in other ways such as by providing source code, reproducible builds, formal verification etc. The exact mechanisms are out of scope of this document.

## Freshness Guarantees {#freshness-guarantees}

<cref> TODO: Discuss freshness guarantees provided by the Attestation Binder.
Differences between Background Check and Passport mode.
</cref>

# Privacy Considerations {#priv-cons}

In this section, we are assuming that the Attester is a TLS client, representing an individual person.
We are concerned about the potential leakage of privacy sensitive information about that person, such as the correlation of different connections initiated by them.

In background-check mode, the Verifier not only has access to detailed information about the Attester's TCB through Evidence, but it also knows the exact time and the party with whom the secure channel establishment is attempted (i.e., the RP).
The privacy implications are similar to online OCSP {{-ocsp}}.
While the RP may trust the Verifier not to disclose any information it receives, the same cannot be assumed for the Attester, which generally has no prior relationship with the Verifier.
Some ways to address this include:

* Client-side redaction of privacy-sensitive evidence claims,
* Using selective disclosure (e.g., SD-JWT {{-sd-jwt}} with EAT {{-rats-eat}}),
* Co-locating the Verifier role with the RP,
* Utilizing privacy-preserving attestation schemes (e.g., DAA {{-rats-daa}}), or
* Utilizing Attesters manufactured with group identities (e.g., {{FIDO-REQS}}).

The latter two also have the property of hiding the peer's identity from the RP.

Note that the equivalent of OCSP "stapling" involves using a passport topology where the Verifier's involvement is unrelated to the TLS session.

## Server Attestation to Unauthenticated Clients {#server-attester-privacy}

Due to the inherent asymmetry of the TLS handshake, when the Attester acts as the
TLS server it produces attestation before the client has authenticated. As a
result, any unauthenticated client that completes the handshake can read
the Claims carried in the server's Evidence, without its own trustworthiness first
being established by the server. The following considerations bound the impact of
this exposure and offer mitigations.

* Passport model with selective disclosure: In the Passport topology
  ({{Section 5.1 of -rats-arch}}), the server presents a Verifier-signed
  Attestation Result instead of the Evidence, so the Evidence never reaches the
  client. As the signer, the Verifier can issue this result in
  selectively-disclosable form (SD-CWT {{-sd-cwt}} or SD-JWT {{-sd-jwt}}), letting
  the server reveal only a subset of Claims.

See {{-rats-privacy}} for a broader treatment of privacy in the RATS context.

# IANA Considerations

## TLS Extensions

IANA is asked to allocate a new TLS extension, `remoteAttestation`, from the
"TLS ExtensionType Values" subregistry of the "Transport Layer Security (TLS)
Extensions" registry {{TLS-Ext-Registry}}.  This extension is used in the
ClientHello, EncryptedExtensions, CertificateRequest, and Certificate messages.
The values carried in this extension are defined in
{{figure-remote-attestation-extension}}.

## TLS Alerts {#tls-alerts}


IANA is requested to allocate values in the "TLS Alerts"
subregistry of the "Transport Layer Security (TLS) Parameters" registry
{{TLS-Param-Registry}} and populate it with the following entries:

- Value: TBD1
- Description: unsupported_attestation_schemes
- DTLS-OK: Y
- Reference: [This document]
- Comment:

- Value: TBD2
- Description: attestation_required
- DTLS-OK: Y
- Reference: [This document]
- Comment:

- Value: TBD3
- Description: attestation_failed
- DTLS-OK: Y
- Reference: [This document]
- Comment:

# Acknowledgements {#acknowledgements}

We would like to thank Paul Howard, Arto Niemi, and Hannes Tschofenig for their contributions to earlier versions of this document.

--- back

# Document History {#document-history}

## draft-fossati-seat-early-attestation-06

* Add a Security Considerations subsection on relay resistance, showing that
  transcript binding (`ClientHello...ServerHello`) and EKM binding offer equal
  relay resistance (see {{relay-resistance}}).
* Expand Privacy Considerations to cover server attestation to unauthenticated
  clients, with mitigations (selective disclosure via
  SD-CWT, and the Passport model) (see {{server-attester-privacy}}).
* Add an appendix with the design rationale for anchoring the binder to the
  transcript rather than to an exporter secret (see {{transcript-vs-exporter}}).
* Add an appendix noting that the `ClientHello...ServerHello` transcript can be
  computed using existing TLS-stack APIs, requiring no new interface (see
  {{transcript-apis}}).
* Clarify that `HKDF-Expand-Label` is the TLS 1.3 wrapper over `HKDF-Expand`
  ({{-hkdf}}) and that its use does not modify the TLS protocol or the key
  schedule.

## draft-fossati-seat-early-attestation-05

* Change extension model to a single `remoteAttestation` extension that covers all handshake messages which need extending.

## draft-fossati-seat-early-attestation-04

* Register the `attestation_failed` alert for Evidence verification failure after
  the `attestation` extension is processed; clarify roles of the three attestation-related
  alerts in {{tls-alerts}}.
* Hash TLS public keys in `HKDF-Expand-Label` context so `HkdfLabel` stays
  within the 255-octet limit (post-quantum public keys); see {{crypto-ops}}.
* Simplify attestation binder derivation to a single shared transcript
  checkpoint (`ClientHello...ServerHello`) for both peers (see {{crypto-ops}}).
* Replaced Derive-Secret with HKDF-Expand-Label

## draft-fossati-seat-early-attestation-03
* Replace the Attestation message by an Attestation (certificate) extension,
to bring this protocol within the requirements of the SEAT charter.
* Define the attestation binder and decouple it from the TLS key schedule.
* List multiple design options for reattestation.
* Add architecture diagram for TLS stack interface with the TEE.
* Add defense-in-depth guidance for measuring TEE, TLS stack, and shim.
* Remove various outdated sections.

## draft-fossati-seat-early-attestation-02

* Fix typo in key schedule. Clarify (again) that this is only adding to the schedule, not modifying any existing key derivations.

## draft-fossati-seat-early-attestation-01

(Submitted by mistake.)

## draft-fossati-seat-early-attestation-00

Initial version of draft-fossati-seat-early-attestation.

This version represents a major architectural change from {{-old-draft}}.
The key changes include:

- Removed certificate extension mechanism for conveying attestation Evidence
- Introduced new `Attestation` handshake message for carrying CMW (Conceptual Message Wrapper) payload
- `Attestation` message sent after CertificateVerify when server is attester
- `Attestation` message sent after CertificateVerify message when client is attester
- Removed use cases section
- Removed KAT (Key Attestation Token) and PAT (Platform Attestation Token) references, using CMW directly
- Nonces (client and server) and attester's TLS identity public key are included in TEE-signed Evidence/AttestationResults within CMW
- CertificateVerify remains unchanged from baseline TLS (no proof-of-possession needed)
- Added session resumption discussion (resumption MUST be rejected if reattestation is required per local policy)
- Added reattestation

<!-- Start of Appendices -->

# Design Rationale: Why the Transcript and Not the Exporter {#transcript-vs-exporter}

{{relay-resistance}} establishes that binding to the
`ClientHello...ServerHello` transcript checkpoint and binding to an exported
value (EKM) offers two independent methods to provide equal relay resistance. This appendix discusses the rationale behind anchoring the
attestation binder to the transcript rather than to an exporter
secret.

* Attestation happens during the handshake. When Evidence is produced and bound
  as part of the handshake, EKM does not yet exist: it is derived only after the
  handshake completes. Evidence produced during the handshake therefore cannot be
  anchored to EKM.

* TLS 1.3 defines an `early_exporter_secret` ({{Section 7.5 of -tls13}}),
  which is available earlier. However, it is only meaningful when a PSK is in use.
  With no PSK, the Early Secret is HKDF-Extract(0, 0), so the early_exporter_secret
  has no secret input; moreover, it is derived from the ClientHello alone and
  carries no contribution from the server side of the handshake. It therefore does
  not provide the two-sided uniqueness ({{terminology}}) the binder requires.

# Computing the Handshake Transcript with Existing TLS APIs {#transcript-apis}

The attestation binder is computed over `Transcript-Hash(ClientHello...ServerHello)`
(see {{crypto-ops}}). Both messages are already held by the TLS stack at the point
attestation runs, so computing the binder requires no change to the TLS
protocol.  Additionally, TLS stacks typically expose handshake messages via callback
interfaces before the handshake completes; the application can obtain ClientHello
and ServerHello through these existing hooks without any new protocol interface.


