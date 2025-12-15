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
  I-D.ietf-tls-extended-key-update: eku
informative:
  RFC6960: ocsp
  RFC9334: rats-arch
  I-D.fossati-tls-attestation: old-draft
  I-D.ietf-rats-eat: rats-eat
  I-D.ietf-rats-daa: rats-daa
  I-D.ietf-oauth-selective-disclosure-jwt: sd-jwt
  I-D.ietf-teep-architecture: teep-arch
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
Such an assurance can be achieved using attestation which is a process by which an entity produces Evidence about itself that another party can use to appraise whether that entity is found in a secure state.
This document describes a protocol extension to the TLS 1.3 handshake that enables the binding of the TLS authentication key to a remote attestation session.
This enables an entity capable of producing attestation Evidence, such as a confidential workload running in a Trusted Execution Environment (TEE), or an IoT device that is trying to authenticate itself to a network access point, to present a more comprehensive set of security metrics to its peer.
These extensions have been designed to allow the peers to use any attestation technology, in any remote attestation topology, and to use them mutually.

--- middle

#  Introduction

Attestation {{-rats-arch}} is the process by which an entity produces evidence about itself that another party can use to evaluate the trustworthiness of that entity.
This document describes a series of protocol extensions to the TLS 1.3 handshake that enables the binding of the TLS authentication key to a remote attestation session.
This enables an attester, such as a confidential workload running in a Trusted Execution Environment (TEE) {{-teep-arch}}, or an IoT device that is trying to authenticate itself to a network access point, to present a more comprehensive set of security metrics to its peer.
This, in turn, allows for the implementation of authorization policies at the relying parties that are based on stronger security signals.

Given the variety of deployed and emerging attestation technologies (e.g., {{TPM1.2}}, {{TPM2.0}}, {{-rats-eat}}) these extensions have been explicitly designed to be agnostic to the attestation formats.
This is achieved by reusing the generic encapsulation defined in {{-cmw}} for transporting Evidence and Attestation Results payloads in the TLS Attestation handshake message.

This specification provides both one-way (server-only) and mutual (client and server) authentication using traditional TLS authentication combined with attestation, and allows the attestation topologies at each peer to be independent of each other.
The proposed design supports both background-check and passport topologies, as described in {{Sections 5.2 and 5.1 of -rats-arch}}.
This is detailed in {{evidence-extensions}} and {{attestation-results-extensions}}.

The protocol we propose is implemented completely at the TLS level, resulting in several related advantages:

* Implementation is within a single system component.
* Security does not depend on application-level code, which tends to be less secure than widely shared infrastructure components.
* It is easier to reason about the application's security, since the peers' identities and security postures are known as soon as the handshake completes
and the TLS connection is established.
* Application code does not need to change. At most, some configuration is needed, similar to the current use of certificate trust stores.

This document does not mandate any particular attestation technology.
Companion documents are expected to define specific attestation mechanisms.

# Conventions and Terminology

The reader is assumed to be familiar with the vocabulary and concepts defined in
{{Section 4 of -rats-arch}}.

The following terms are used in this document:

{: vspace="0"}

TLS Identity Key (TIK):
: A cryptographic key used by one of the peers to authenticate itself during the
TLS handshake. The protocol's security is critically dependent on the provenance, lifetime and
protection properties of the TIK. The TIK MUST be the X.509 certificate's end entity key and is maintained and protected by the TEE.

TIK-C, TIK-S:
: The TIK that identifies the client or the server, respectively.

TIK-C-ID, TIK-S-ID:

: An identifier for TIK-C or respectively, TIK-S. This may be a fingerprint
(cryptographic hash) of the public key, but other implementations are possible.


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

- Negotiation extensions: New TLS extensions are added to ClientHello and
  EncryptedExtensions messages to negotiate the use of attestation and indicate
  supported attestation formats and verifiers.

- Independent handshake message: A new `Attestation` handshake message is
  introduced that carries attestation Evidence or Attestation Results. This message
  is completely independent of the standard TLS handshake flow and does not
  interfere with existing handshake messages or their processing.

- Independent key derivation: Key derivation for attestation (see {{crypto-ops}}) ensures independence of the regular TLS key schedule. As a result, attestation
  processing does not affect the standard TLS key derivation and security properties.

This minimal integration approach provides intuitive reasoning why TLS security is
not adversely affected by the addition of attestation. The attestation components
operate independently and do not modify the core TLS handshake protocol or key
derivation mechanisms. However, formal validation of these security properties is
still needed.

# Attestation Extensions

As typical with new features in TLS, the client indicates support for the new
extension in the ClientHello message. The newly introduced extensions allow
attestation Evidence or Attestation Results to be exchanged. Freshness of the
exchanged Evidence is guaranteed through secret derivation from the TLS main
secret and message transcript (see {{crypto-ops}}) when the Background Check
Model is in use. In the Passport Model, freshness expectations are more relaxed
and are governed by the lifetime of the signed Attestation Results.

When either the Evidence or the Attestation Results extension is successfully
negotiated, attestation Evidence or Attestation Results are conveyed in an
`Attestation` handshake message (see {{attestation-message-section}}). The
CMW payload in the Attestation message contains the attestation Evidence or
Attestation Results encoded according to {{-cmw}}.

The attestation payload MUST contain assertions relating to the attester's TLS
Identity Key (TIK-C for client attester, TIK-S for server attester), which
associate the private key with the attestation information. The TEE's signature
over the Evidence or AttestationResults within the CMW MUST include a secret derived
from the TLS main secret and the message transcript up to ServerHello (see {{crypto-ops}})
and the attester's TLS identity public key, as specified in {{attestation-message-section}}.

The relying party can obtain and appraise the remote Attestation Results either
directly from the Attestation message (in the Passport Model), or by relaying
the Evidence from the Attestation message to the Verifier and receiving the
Attestation Results. Subsequently, the attested key is used to verify the
CertificateVerify message, which remains unchanged from baseline TLS.

When using the Passport Model, the remote Attestation Results obtained by the
attester from its trusted Verifiers can be cached and used for any number of
subsequent TLS handshakes, as long as the freshness policy requirements are
satisfied.

In TLS a client has to demonstrate possession of the private key via the
CertificateVerify message, when client-based authentication is requested.
This behavior remains unchanged in the current protocol, with the CertificateVerify
message proving possession of the TIK.

This protocol supports both monolithic and split implementations. In a monolithic
implementation, the TLS stack is completely embedded within the TEE. In a split
implementation, the TLS stack is located outside the TEE, but any private keys
(and in particular, the TIK) only exist within the TEE. In order to support
both options, only the TIK's identity, its public component and a short generated secret are ever
passed between the Client or Server TLS stack and its Attestation Service.
While the two types of implementations offer identical functionality,
their security properties often differ, see {{sec-guarantees}} for more details.

## Attestation Handshake Message {#attestation-message-section}

When attestation is negotiated via the extensions defined in this document,
attestation Evidence or Attestation Results are conveyed in a new handshake
message type: `Attestation`. This message carries a CMW (Conceptual Message
Wrapper) payload as defined in {{-cmw}}.

The `Attestation` message structure is defined as follows:

~~~~
    enum {
        /* other handshake message types defined in {{I-D.ietf-tls-rfc8446bis}} */
        attestation(TBD),
        (255)
    } HandshakeType;

    struct {
        HandshakeType msg_type;    /* handshake type */
        uint24 length;             /* bytes in message */
        select (Handshake.msg_type) {
            case attestation:
                Attestation;
            /* other handshake message types */
        };
    } Handshake;

    struct {
        opaque cmw_payload<1..2^24-1>;
    } Attestation;
~~~~
{: #figure-attestation-message title="Attestation Handshake Message Structure."}

The `cmw_payload` field contains a CMW structure as defined in {{-cmw}}.
Both JSON and CBOR serializations are allowed in CMW, with the emitter choosing
which serialization to use.

The CMW payload MUST contain attestation Evidence (in Background Check Model)
or Attestation Results (in Passport Model) that binds the TLS Identity Key (TIK)
to the platform and workload state. The TEE's signature over the Evidence or
AttestationResults within the CMW MUST include:

- A secret derived from the TLS main secret and the message transcript, up to ServerHello,
ensuring freshness of the attestation.
- The attester's TLS identity public key (TIK-C for client attester, TIK-S for
  server attester)

This binding ensures that the attested key is the one used in the TLS handshake
and provides freshness guarantees through secret derivation. See {{crypto-ops}} for details.

# Use of Attestation in the TLS Handshake

For both the Passport Model (described in section 5.1 of {{RFC9334}}) and
Background Check Model (described in Section 5.2 of {{RFC9334}}) the following
modes of operation are allowed when used with TLS, namely:

- TLS client is the attester,
- TLS server is the attester, and
- TLS client and server mutually attest towards each other.

We will show the message exchanges of the first two cases in sub-sections below.
Mutual authentication via attestation combines these two (non-interfering)
flows, including cases where one of the peers uses the Passport Model for its
attestation, and the other uses the Background Check Model.

## Handshake Overview {#handshake-overview}

The handshake defined here is analogous to certificate-based authentication in a regular TLS handshake.
The peer being attested first proves possession of the private key using the CertificateVerify message, which remains unchanged from baseline TLS. Following that, the TLS Identity Key (TIK) is attested by the TEE, with attestation being carried in a new Attestation handshake message (see {{attestation-message-section}}).

The attestation Evidence or Attestation Results are conveyed in an `Attestation`
handshake message (see {{attestation-message-section}}), which carries a CMW
payload as defined in {{-cmw}}.

## TLS Client Authenticating Using Evidence

In this use case, the TLS server (acting as a relying party) challenges the TLS
client (as the attester) to provide Evidence. A session-specific value is derived
(see {{crypto-ops}})
which incorporates randomness from both client and server, and this value is fed into the generation
of the Evidence.
The
client sends the Evidence in an `Attestation` handshake message after the
`CertificateVerify` message. The TLS server, when receiving the Evidence, will have
to contact the Verifier (which is not shown in the diagram).

An example of this flow can be found in device onboarding where the
client initiates the communication with cloud infrastructure to
get credentials, firmware and other configuration data provisioned
to the device. For the server to consider the device genuine it needs
to present Evidence.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + evidence_proposal
     | + key_share*
     | + signature_algorithms*
     v                         -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                                               v
                                        {EncryptedExtensions}  ^  Server
                                          + evidence_proposal  |  Params
                                         {CertificateRequest}  v
                                               {Certificate}  ^
                                         {CertificateVerify}  |
                                         {Attestation}        | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate}
Auth | {CertificateVerify}
     | {Attestation}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-background-check-model1 title="TLS Client Providing Evidence to TLS Server."}


## TLS Server Authenticating Using Evidence

In this use case the TLS client challenges the TLS server to present Evidence.
The TLS server acts as an attester while the TLS client is the relying party.
The server sends the Evidence in an `Attestation` handshake message after the
`CertificateVerify` message. The TLS client, when receiving the Evidence,
will have to contact the Verifier (which is not shown in the diagram).

An example of this flow can be found in confidential computing where
a compute workload is only submitted to the server infrastructure
once the client/user is assured that the confidential computing platform is
genuine.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + evidence_request
     | + key_share*
     | + signature_algorithms*
     v                         -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                                               v
                                        {EncryptedExtensions}  ^  Server
                                          + evidence_request   |  Params
                                         {CertificateRequest}  v
                                               {Certificate}  ^
                                         {CertificateVerify}  |
                                         {Attestation}         | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate}
Auth | {CertificateVerify}
     | {Attestation}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-background-check-model2 title="TLS Server Providing Evidence to TLS Client."}

## TLS Client Authenticating Using Attestation Results

In this use case the TLS client, as the attester, provides Attestation Results
to the TLS server. The TLS client is the attester and the TLS server acts as
a relying party. Prior to delivering its Certificate message, the client must
contact the Verifier (not shown in the diagram) to receive the Attestation
Results that it will use as credentials. The client sends the Attestation
Results in an `Attestation` handshake message after the `CertificateVerify` message.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + results_proposal
     | + key_share*
     | + signature_algorithms*
     v                         -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                                               v
                                        {EncryptedExtensions}  ^  Server
                                           + results_proposal  |  Params
                                         {CertificateRequest}  v
                                               {Certificate}  ^
                                         {CertificateVerify}   |
                                         {Attestation}         | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate}
Auth | {CertificateVerify}
     | {Attestation}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-passport-model1 title="TLS Client Providing Results to TLS Server."}


## TLS Server Authenticating Using Attestation Results

In this use case the TLS client, as the relying party, requests Attestation
Results from the TLS server. Prior to delivering its Certificate message, the
server must contact the Verifier (not shown in the diagram) to receive the
Attestation Results that it will use as credentials. The server sends the
Attestation Results in an `Attestation` handshake message after the
`CertificateVerify` message.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + results_request
     | + key_share*
     | + signature_algorithms*
     v                         -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                                               v
                                        {EncryptedExtensions}  ^  Server
                                           + results_request   |  Params
                                         {CertificateRequest}  v
                                               {Certificate}  ^
                                         {CertificateVerify}   |
                                         {Attestation}         | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate}
Auth | {CertificateVerify}
     | {Attestation}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-passport-model2 title="TLS Server Providing Attestation Results to TLS Client."}

## Cryptographic Operations {#crypto-ops}

This section defines the key derivation for attestation, which operates independently
from the regular TLS key schedule as described in {{Section 7.1 of I-D.ietf-tls-rfc8446bis}}.

The attestation key derivation uses HKDF {{Section 7.1 of I-D.ietf-tls-rfc8446bis}} to derive
attestation-specific secrets from the TLS main secret. Two attestation main
secrets are derived: one for the client (`c_attestation_main`) and one for the
server (`s_attestation_main`).

The key derivation follows this structure:

~~~~
   0
   |
   v
(EC)DHE -> HKDF-Extract = Handshake Secret
   |
   v
Derive-Secret(., "derived secret", "")
   |
   v
0 -> HKDF-Extract = Master Secret
   |
   +-> Derive-Secret(., "c attestation master", ClientHello...ServerHello)
   |                     = c_attestation_main
   |
   +-> Derive-Secret(., "s attestation master", ClientHello...ServerHello)
   |                     = s_attestation_main
~~~~
{: #figure-attestation-key-schedule title="Attestation Key Schedule."}

The attestation main secrets (`c_attestation_main` and `s_attestation_main`)
are derived from the TLS main secret using Derive-Secret as defined in
{{Section 7.1 of I-D.ietf-tls-rfc8446bis}}, with the labels "c attestation master" and
"s attestation master" respectively, and the handshake transcript up to and
including ServerHello as the context.

The client's attestation secret (`c_attestation_secret`) that will be signed by
the TEE is derived by applying HKDF-Expand-Label to `c_attestation_main` with
the label "attestation" and the client's TLS public key as the context:

~~~~
c_attestation_secret = HKDF-Expand-Label(c_attestation_main, "Early Attestation",
                                         TLS_Client_Public_Key, Hash.length)
~~~~

Similarly, the server's attestation secret (`s_attestation_secret`) is derived
from `s_attestation_main`:

~~~~
s_attestation_secret = HKDF-Expand-Label(s_attestation_main, "Early Attestation",
                                         TLS_Server_Public_Key, Hash.length)
~~~~

This ensures that each attestation secret is bound to the specific TLS public
key being attested.

## The TLS Stack's Interface to the TEE

When the TEE signs the Evidence or Attestation Results, it also binds them to the TLS Identity public key and the TLS
session. TEE implementations differ, and some only allow a single nonce to be added to the signature with no associated checks.
Therefore we adopt a defense-in-depth approach:

* TEE wrapper libraries and TLS stacks SHOULD NOT allow direct access to the Evidence/AR generation API without checking the
nonce. At the very least, the nonce SHOULD be the result of HKDF with an allowlist of labels.
* The RP SHOULD NOT base its trust decision only on the identity of the issuer of the Evidence's (or AR's) trust root. It SHOULD also
ensure that the software layer above it is endorsed.
* The TEE itself, when possible, SHOULD generate the nonce by running HKDF with an allowlisted label and if it holds the TIK, SHOULD
validate the pubic key.

# DTLS Considerations

The Attestation message MUST be handled using the existing DTLS handshake mechanisms for fragmentation, ordering, and retransmission to ensure reliable delivery.

In DTLS, handshake messages that do not solicit a response are acknowledged using the DTLS ACK message. Because the Attestation handshake message does not elicit a response, the receiving peer MUST send a DTLS ACK upon receipt of the Attestation message. This ACK confirms only that the message was received; it does not indicate that attestation appraisal has completed.

Once the attester receives the ACK, it MUST stop retransmitting the Attestation message. The receiving peer performs attestation appraisal asynchronously and applies its authorization policy once appraisal results become available.

# After The Initial Handshake {#after-handshake}

This section covers protocol behavior after the initial handshake, including
session resumption, reattestation and the interaction between them.

## Session Resumption {#session-resumption}

TLS 1.3 supports session resumption using Pre-Shared Keys (PSK) as defined in
{{Section 4.6 of I-D.ietf-tls-rfc8446bis}}. When using attestation, session resumption works
normally when reattestation is not required.

If client reattestation is required according to local policy (e.g., based on timing
since the last attestation or changes in attestation state), session resumption
MUST be rejected. The decision to reject resumption is per local policy and may
depend on the timing of the resumption attempt relative to the required
reattestation period. When resumption is rejected, the client MUST
initiate a full handshake with attestation to obtain fresh attestation Evidence
or Attestation Results.

The rationale for rejecting resumption when reattestation is required is that
attestation state may have changed since the original handshake, and fresh
verification is needed to ensure the peer's platform and workload remain in a
trustworthy state. If the client wishes to retain a long-running connection, it SHOULD
perform reattestation {{reattestation}} periodically, as per local policy.

## Reattestation {#reattestation}

Over time, attestation Evidence or Attestation Results may become stale and
require refresh. Long-lived TLS connections require updated assurance that
the peer continues to operate in a trustworthy state. This document
therefore supports reattestation, in which either peer MAY request fresh
Evidence at any time post-handshake. The attester MUST generate evidence
using a freshly derived attestation_secret.

Reattestation is tied to the completion of an Extended Key Update (EKU) exchange {{!I-D.ietf-tls-extended-key-update}}. TLS peers that require reattestation MUST support EKU,
since reattestation depends on the key schedule update defined in the EKU draft.
The first two messages of an EKU exchange introduce fresh key-exchange input and
make `Main Secret N+1` available to both peers.

The Attestation message MUST be sent immediately before the attestor sends
its EKU(new_key_update) message. Once `Main Secret N+1` is available
(after the first two EKU messages), the attester derives a new
attestation_secret from `Main Secret N+1`, using the concatenation of the
EKU request and response messages and its TLS identity public key as context.

The receiving peer, however, MUST NOT process the Attestation until the
EKU exchange and the authenticated transition step have completed. This
ensures that attestation bound to `Main Secret N+1` is accepted only after
both peers have confirmed that they share the same updated key state.

For a client attester:

~~~
client_attestation_secret =
      Derive-Secret(Main Secret N+1,
                    "reattestation",
                    EKU(request) ||
                    EKU(response) ||
                    TLS_Client_Public_Key)
~~~

For a server attester:

~~~
server_attestation_secret =
      Derive-Secret(Main Secret N+1,
                    "reattestation",
                    EKU(request) ||
                    EKU(response) ||
                    TLS_Server_Public_Key)
~~~

Including the EKU request and response messages ensures that the resulting attestation secret
is bound to the specific EKU exchange and therefore reflects fresh key-exchange entropy
introduced by EKU.

After deriving the fresh attestation_secret, the attester:

1. generates fresh Evidence using the new attestation_secret and
2. sends a new `Attestation` handshake message containing the updated CMW payload.

The TLS peer validates that the attestation payload incorporates the newly derived attestation secret.

Reattestation uses the Attestation formats that were negotiated during the initial handshake,
there is no re-negotiation at this stage.

The decision to initiate reattestation is per local policy and may be based on
factors such as elapsed time since the last attestation, changes in platform
state, or security policy requirements.

# Negotiating This Protocol {#negotiating-protocol}

This section defines the TLS extensions used to negotiate the use of attestation
in the TLS handshake. Two models are supported: the Background Check Model, where
Evidence is exchanged and verified during the handshake, and the Passport Model,
where pre-verified Attestation Results are presented. The extensions defined
here allow peers to indicate their support for attestation and negotiate which
attestation format and Verifier to use.

<cref>Can we simplify this structure: remove the dual request/proposal, and unify the evidence+AR to a single
negotiation extension. But also express Passport mode with and without freshness.</cref>

## Evidence Extensions (Background Check Model) {#evidence-extensions}

The EvidenceType structure contains an indicator for the type of Evidence
expected in the `Attestation` handshake message. The Evidence contained in
the CMW payload is sent in the `Attestation` handshake message (see {{attestation-message-section}}).

~~~~
    enum { CONTENT_FORMAT(0), MEDIA_TYPE(1) } typeEncoding;

    struct {
        typeEncoding type_encoding;
        select (EvidenceType.type_encoding) {
            case CONTENT_FORMAT:
                uint16 content_format;
            case MEDIA_TYPE:
                opaque media_type<0..2^16-1>;
        };
    } EvidenceType;

    struct {
        select(Handshake.msg_type) {
            case client_hello:
                EvidenceType supported_evidence_types<1..2^8-1>;
            case server_hello:
            case encrypted_extensions:
                EvidenceType selected_evidence_type;
        }
    } evidenceRequestTypeExtension;

    struct {
        select(Handshake.msg_type) {
            case client_hello:
                EvidenceType supported_evidence_types<1..2^8-1>;
            case server_hello:
            case encrypted_extensions:
                EvidenceType selected_evidence_type;
        }
    } evidenceProposalTypeExtension;
~~~~
{: #figure-extension-evidence title="TLS Extension Structure for Evidence."}

Values for media_type are defined in {{iana-media-types}}.
Values for content_format are defined in {{iana-content-formats}}.

## Attestation Results Extensions (Passport Model) {#attestation-results-extensions}

~~~~
    struct {
        opaque verifier_identity<0..2^16-1>;
    } VerifierIdentityType;

    struct {
        select(Handshake.msg_type) {
            case client_hello:
                VerifierIdentityType trusted_verifiers<1..2^8-1>;

            case server_hello:
            case encrypted_extensions:
                VerifierIdentityType selected_verifier;
        }
    } resultsRequestTypeExtension;

    struct {
        select(Handshake.msg_type) {
            case client_hello:
                VerifierIdentityType trusted_verifiers<1..2^8-1>;

            case server_hello:
            case encrypted_extensions:
                VerifierIdentityType selected_verifier;
        }
    } resultsProposalTypeExtension;
~~~~
{: #figure-extension-results title="TLS Extension Structure for Attestation Results."}

In the Passport Model, Attestation Results are sent in an `Attestation` handshake
message (see {{attestation-message-section}}) containing a CMW structure. The CMW
structure is defined in {{-cmw}}.

# TLS Client and Server Handshake Behavior {#behavior}

The high-level message exchange in {{figure-overview}} shows the
evidence_proposal, evidence_request, results_proposal, and results_request
extensions added to the ClientHello and the EncryptedExtensions messages.

~~~~
       Client                                           Server

Key  ^ ClientHello
Exch | + key_share*
     | + signature_algorithms*
     | + psk_key_exchange_modes*
     | + pre_shared_key*
     | + evidence_proposal*
     | + evidence_request*
     | + results_proposal*
     v + results_request*
     -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                            + pre_shared_key*  v
                                        {EncryptedExtensions}  ^  Server
                                         + evidence_proposal*  |
                                          + evidence_request*  |
                                          + results_proposal*  |
                                           + results_request*  |  Params
                                        {CertificateRequest*}  v
                                               {Certificate*}  ^
                                         {CertificateVerify*}  |
                                         {Attestation*}        | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate*}
Auth | {CertificateVerify*}
     | {Attestation*}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]
~~~~
{: #figure-overview title="Attestation Message Overview."}

## Background Check Model

### Client Hello

To indicate the support for passing Evidence in TLS following the
Background Check Model, clients include the evidence_proposal
and/or the evidence_request extensions in the ClientHello.

The evidence_proposal extension in the ClientHello message indicates
the Evidence types the client is able to provide to the server.

The evidence_request extension in the ClientHello message indicates
the Evidence types the client challenges the server to
provide in an `Attestation` handshake message.

The evidence_proposal and evidence_request extensions sent in
the ClientHello each carry a list of supported Evidence types,
sorted by preference.  When the client supports only one Evidence
type, it is a list containing a single element.

The client MUST omit Evidence types from the evidence_proposal
extension in the ClientHello if it cannot respond to a request
from the server to present a proposed Evidence type, or if
the client is not configured to use the proposed Evidence type
with the given server.  If the client has no Evidence types
to send in the ClientHello it MUST omit the evidence_proposal
extension in the ClientHello.

The client MUST omit Evidence types from the evidence_request
extension in the ClientHello if it is not able to pass the
indicated verification type to a Verifier.  If the client does
not act as a relying party with regards to Evidence processing
(as defined in the RATS architecture) then the client MUST
omit the evidence_request extension from the ClientHello.

### Server Hello

If the server receives a ClientHello that contains the
evidence_proposal extension and/or the evidence_request
extension, then three outcomes are possible:

-  The server does not support the extensions defined in this
   document.  In this case, the server returns the EncryptedExtensions
   without the extensions defined in this document.

-  The server supports the extensions defined in this document, but
   it does not have any Evidence type in common with the client.
   Then, the server terminates the session with a fatal alert of
   type "unsupported_evidence".

-  The server supports the extensions defined in this document and
   has at least one Evidence type in common with the client.  In
   this case, the processing rules described below are followed.

The evidence_proposal extension in the ClientHello indicates
the Evidence types the client is able to provide to the server.  If the
server wants to request Evidence from the client, it MUST include the
evidence_proposal extension in the EncryptedExtensions. This
evidence_proposal extension in the EncryptedExtensions then indicates
what Evidence format the client is requested to provide in an
`Attestation` handshake message sent after the `CertificateVerify` message.
The Evidence contained in the CMW payload MUST include a secret derived from
the TLS main secret and the message transcript up to ServerHello (see {{crypto-ops}})
in the TEE's signature, along with the client's TLS identity public key (TIK-C).
The value conveyed in the evidence_proposal extension by the server MUST be
selected from one of the values provided in the evidence_proposal extension
sent in the ClientHello.

If none
of the Evidence types supported by the client (as indicated in the
evidence_proposal extension in the ClientHello) match the
server-supported Evidence types, then the evidence_proposal
extension in the ServerHello MUST be omitted.

The evidence_request extension in the ClientHello indicates what
types of Evidence the client can challenge the server to return
in an `Attestation` handshake message. With the evidence_request
extension in the EncryptedExtensions, the server indicates the
Evidence type carried in the `Attestation` handshake message sent
after the CertificateVerify by the server. The Evidence
contained in the CMW payload MUST include a secret derived from
the TLS main secret and the message transcript up to ServerHello (see {{crypto-ops}})
in the TEE's signature, along with
the server's TLS identity public key (TIK-S).
The Evidence type in the evidence_request extension MUST contain
a single value selected from the evidence_request extension in
the ClientHello.

## Passport Model

The `results_proposal` and `results_request` extensions are used to negotiate
the protocol defined in this document, and in particular to negotiate the Verifier identities supported by each peer. These
extensions are included in the ClientHello and ServerHello messages.

### Client Hello

To indicate the support for passing Attestation Results in TLS following the
Passport Model, clients include the results_proposal and/or the results_request
extensions in the ClientHello message.

The results_proposal extension in the ClientHello message indicates the Verifier
identities from which the client can relay Attestation Results. The client sends the Attestation Results in an
`Attestation` handshake message after the `CertificateVerify` message.

The results_request extension in the ClientHello message indicates the Verifier
identities from which the client expects the server to provide Attestation
Results in an `Attestation` handshake message sent after the CertificateVerify.

The results_proposal and results_request extensions sent in the ClientHello each
carry a list of supported Verifier identities, sorted by preference.  When the
client supports only one Verifier, it is a list containing a single element.

The client MUST omit Verifier identities from the results_proposal extension in
the ClientHello if it cannot respond to a request from the server to present
Attestation Results from a proposed Verifier, or if the client is not configured
to relay the Results from the proposed Verifier with the given server. If the
client has no Verifier identities to send in the ClientHello it MUST omit the
results_proposal extension in the ClientHello.

The client MUST omit Verifier identities from the results_request extension in
the ClientHello if it is not configured to trust Attestation Results issued by
said verifiers. If the client does not act as a relying party with regards to
the processing of Attestation Results (as defined in the RATS architecture) then
the client MUST omit the results_request extension from the ClientHello.

### Server Hello

If the server receives a ClientHello that contains the results_proposal
extension and/or the results_request extension, then three outcomes are
possible:

-  The server does not support the extensions defined in this document.  In this
   case, the server returns the EncryptedExtensions without the extensions
   defined in this document.

-  The server supports the extensions defined in this document, but it does not
   have any trusted Verifiers in common with the client. Then, the server
   terminates the session with a fatal alert of type "unsupported_verifiers".

-  The server supports the extensions defined in this document and has at least
   one trusted Verifier in common with the client.  In this case, the processing
   rules described below are followed.

The results_proposal extension in the ClientHello indicates the Verifier
identities from which the client is able to provide Attestation Results to the
server.  If the server
wants to request Attestation Results from the client, it MUST include the
results_proposal extension in the EncryptedExtensions. This results_proposal
extension in the EncryptedExtensions then indicates what Verifier the client is
requested to provide Attestation Results from in an `Attestation` handshake
message sent after the `CertificateVerify` message.  The value conveyed in the
results_proposal extension by the server MUST be selected from one of the
values provided in the results_proposal extension sent in the ClientHello.

If none of the
Verifier identities proposed by the client (as indicated in the results_proposal
extension in the ClientHello) match the server-trusted Verifiers, then the
results_proposal extension in the ServerHello MUST be omitted.

The results_request extension in the ClientHello indicates what Verifiers the
client trusts as issuers of Attestation Results for the server. With the
results_request extension in the EncryptedExtensions, the server indicates the
identity of the Verifier who issued the Attestation Results carried in the
`Attestation` handshake message sent after the CertificateVerify by the
server. The Verifier identity in the results_request extension MUST contain a
single value selected from the results_request extension in the ClientHello.

# Security Considerations {#sec-cons}

TBD.

## Security Guarantees {#sec-guarantees}

We note that as a pure cryptographic protocol, attested TLS as-is only guarantees that the Identity Key is known by the TEE. A number of additional guarantees must be provided by the platform and/or the TLS stack,
and the overall security level depends on their existence and quality of assurance:

* The Identity Key is generated by the TEE.
* The Identity Key is never exported or leaked outside the TEE.
* The TLS protocol, whether implemented by the TEE or outside the TEE, is implemented correctly and (for example) does not leak any session key material.

These properties may be explicitly promised ("attested") by the platform, or they can be assured in other ways such as by providing source code, reproducible builds, formal verification etc. The exact mechanisms are out of scope of this document.

## Freshness Guarantees {#freshness-guarantees}

<cref> TODO: Discuss freshness guarantees provided by secret derivation from
the TLS main secret and message transcript. Differences between Background Check and Passport mode.
</cref>

## Security of Reattestation After EKU

Reattestation relies on the assumption that both peers have derived the same
`Main Secret N+1` during the preceding EKU exchange. EKU by itself does not
guarantee that the peers transitioned to a consistent key state in the presence
of an active attacker. Deployments that require stronger guarantees will have use an
authenticated transition mechanism discussed in {{!I-D.ietf-tls-extended-key-update}}
(e.g., post-handshake client authentication or Exported Authenticators) to
detect key-schedule divergence before relying on reattestation results.

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

Due to the inherent asymmetry of the TLS protocol, if the Attester acts as the TLS server, a malicious TLS client could potentially retrieve sensitive information from attestation Evidence without the client's trustworthiness first being established by the server.

# IANA Considerations

## TLS Extensions

IANA is asked to allocate four new TLS extensions, evidence_request,
evidence_proposal, results_request, results_proposal, from the "TLS
ExtensionType Values" subregistry of the "Transport Layer Security (TLS)
Extensions" registry {{TLS-Ext-Registry}}.  These extensions are used in the
ClientHello and the EncryptedExtensions messages. The values carried in these
extensions are taken from TBD.

## TLS Alerts

IANA is requested to allocate a value in the "TLS Alerts"
subregistry of the "Transport Layer Security (TLS) Parameters" registry
{{TLS-Param-Registry}} and populate it with the following entries:

- Value: TBD1
- Description: unsupported_evidence
- DTLS-OK: Y
- Reference: [This document]
- Comment:

- Value: TBD2
- Description: unsupported_verifiers
- DTLS-OK: Y
- Reference: [This document]
- Comment:

## TLS Handshake Message Types

IANA is requested to allocate a new value in the "TLS HandshakeType" registry
of the "Transport Layer Security (TLS) Parameters" registry {{TLS-Param-Registry}},
as follows:

- Value: TBD
- Description: attestation
- DTLS-OK: Y
- Reference: [This document]
- Comment: Used to carry attestation Evidence or Attestation Results in the TLS handshake

# Acknowledgements {#acknowledgements}

We would like to thank Paul Howard, Arto Niemi, and Hannes Tschofenig for their contributions to earlier versions of this document.

--- back

# Document History {#document-history}

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

# Design Rationale {#design-rationale}

This appendix explains the rationale for introducing a dedicated `Attestation`
handshake message, instead of embedding attestation in an extension inside
the TLS `Certificate` message. That approach fails to meet key security,
and privacy requirements.

## Requires Certificate Authentication

TLS 1.3 supports authentication modes where no `Certificate` message is sent:

* PSK-based authentication
* PAKE-based authentication {{!I-D.ietf-tls-pake}}

A design that relies on a `Certificate` message extension cannot operate in
these cases. In contrast, a dedicated `Attestation` handshake message works
regardless of authentication mode, making it compatible with the full TLS
authentication spectrum.

## Reattestation Not Fully Supported

TLS allows Post-Handshake client authentication {{Section 4.2.6 of I-D.ietf-tls-rfc8446bis}}
but provides no mechanism for Post-Handshake server authentication. As a result, a design
that embeds attestation inside the `Certificate` message would allow only the client and
not the server to refresh its attestation. This is insufficient for deployments that
require periodic server reattestation.
