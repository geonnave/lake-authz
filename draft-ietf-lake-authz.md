---
title: Lightweight Authorization using Ephemeral Diffie-Hellman Over COSE (ELA)
docname: draft-ietf-lake-authz-latest
abbrev: ELA
ipr: trust200902
cat: std
submissiontype: IETF
area: Security
workgroup: LAKE Working Group
keyword: Internet-Draft
coding: utf-8
pi: # can use array (if all yes) or hash here
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 2

author:
- name: Göran Selander
  surname: Selander
  org: Ericsson AB
  country: Sweden
  email: goran.selander@ericsson.com
- name: John Preuß Mattsson
  initials: J
  surname: Preuß Mattsson
  org: Ericsson AB
  country: Sweden
  email: john.mattsson@ericsson.com
- name: Mališa Vučinić
  surname: Vučinić
  org: INRIA
  country: France
  email: malisa.vucinic@inria.fr
- name: Geovane Fedrecheski
  surname: Fedrecheski
  org: INRIA
  country: France
  email: geovane.fedrecheski@inria.fr
- name: Michael Richardson
  surname: Richardson
  org: Sandelman Software Works
  country: Canada
  email: mcr+ietf@sandelman.ca


normative:

  RFC8366:
  RFC8392:
  RFC8949:
  RFC9052:
  RFC8613:
  RFC9528:
  NIST-800-56A:
    author:
      -
        ins: E. Barker
        name: Elaine Barker
      -
        ins: L. Chen
        name: Lily Chen
      -
        ins: A. Roginsky
        name: Allen Roginsky
      -
        ins: A. Vassilev
        name: Apostol Vassilev
      -
        ins: R. Davis
        name: Richard Davis
    title: Recommendation for Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography - NIST Special Publication 800-56A, Revision 3
    date: 2018-04
    target: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-56Ar3.pdf

informative:

  RFC2119:
  RFC3172:
  RFC5280:
  RFC6761:
  RFC7228:
  RFC7593:
  RFC8137:
  RFC8174:
  RFC8446:
  RFC8610:
  RFC8615:
  RFC8995:
  RFC9031:
  I-D.ietf-core-oscore-edhoc:
  I-D.ietf-lake-reqs:
  I-D.amsuess-core-coap-over-gatt:
  I-D.ietf-lake-edhoc-impl-cons:
  IEEE802.15.4:
    title: "IEEE Std 802.15.4 Standard for Low-Rate Wireless Networks"
    author:
      ins: "IEEE standard for Information Technology"
  IEEE802.1X:
    title: "IEEE Standard for Local and metropolitan area networks - Port-Based Network Access Control"
    date: February 2010
    author:
      ins: "IEEE standard for Information Technology"

venue:
  group: "Lightweight Authenticated Key Exchange"
  type: "Working Group"
  mail: "lake@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/lake/"
  github: "lake-wg/authz"
  latest: "https://lake-wg.github.io/authz/draft-ietf-lake-authz.html"

--- abstract

Ephemeral Diffie-Hellman Over COSE (EDHOC) is a lightweight authenticated key exchange protocol intended for use in constrained scenarios.
This document specifies Lightweight Authorization using EDHOC (ELA).
The procedure allows authorizing enrollment of new devices using the extension point defined in EDHOC.
ELA is applicable to zero-touch onboarding of new devices to a constrained network leveraging trust anchors installed at manufacture time.

--- middle

# Introduction  {#intro}


For constrained IoT deployments {{RFC7228}} the overhead and processing contributed by security protocols may be significant, which motivates the specification of lightweight protocols that are optimizing, in particular, message overhead (see {{I-D.ietf-lake-reqs}}).
This document describes Lightweight Authorization using EDHOC (ELA), a procedure for augmenting the lightweight authenticated Diffie-Hellman key exchange EDHOC {{RFC9528}} with third party-assisted authorization.

ELA involves a device, a domain authenticator, and an enrollment server.
The device and domain authenticator perform mutual authentication and authorization, assisted by the enrollment server that provides relevant authorization information to the device (a "voucher") and to the authenticator. The high-level model is similar to BRSKI {{RFC8995}}.

In this document, we consider the target interaction for which authorization is needed to be "enrollment", for example joining a network for the first time (e.g., {{RFC9031}}), but it can be applied to authorize other target interactions.

The enrollment server may represent the manufacturer of the device, or some other party with information about the device from which a trust anchor has been pre-provisioned into the device.
The (domain) authenticator may represent the service provider or some other party controlling access to the network in which the device is enrolling.

ELA assumes that authentication between device and authenticator is performed with EDHOC {{RFC9528}}, and defines the integration of a lightweight authorization procedure using the External Authorization Data (EAD) fields defined in EDHOC.

ELA enables a low message count by performing authorization and enrollment in parallel with authentication, instead of in sequence, which is common for network access.
It further reuses protocol elements from EDHOC, leading to reduced message sizes on constrained links.

This protocol is applicable to a wide variety of settings, and can be mapped to different authorization architectures.

## Terminology   {#terminology}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Readers are expected to have an understanding of CBOR {{RFC8949}}, CDDL {{RFC8610}}, and EDHOC {{RFC9528}}.
Appendix C.1 of {{RFC9528}} contains some basic info about CBOR.

# Protocol Outline {#outline}

The goal of ELA is to enable a (potentially constrained) device (U) to enroll into a domain over a constrained link.
The device authenticates and enforces authorization of the (non-constrained) domain authenticator (V) with the help of a voucher conveying authorization information.
The voucher has a similar role as in {{RFC8366}} but should be considerably more compact.
The domain authenticator, in turn, authenticates the device and authorizes its enrollment into the domain.
ELA also enables scenarios where only V needs to authorize U, by allowing the voucher to be optional.

The procedure is assisted by a (non-constrained) enrollment server (W) located in a non-constrained network behind the domain authenticator, e.g., on the Internet, providing information to the device (conveyed in the voucher) and to the domain authenticator as part of the protocol.

The objective of this document is to specify such a protocol that is lightweight over the constrained link, by reusing elements of EDHOC {{RFC9528}} and by shifting message overhead to the non-constrained side of the network.
See illustration in {{fig-overview}}.

Note the cardinality of the involved parties. It is expected that the domain authenticator needs to handle a large unspecified number of devices, but for a given device type or manufacturer it is expected that one or a few nodes host enrollment servers.

~~~~~~~~~~~ aasvg

                Voucher
                Info
+----------+      |     +---------------+           +---------------+
|          +------+---->|               |  Voucher  |               |
|          |<-----+-----+    Domain     |  Request  |   Enrollment  |
|  Device  +------o---->| Authenticator +---------->|     Server    |
|          |<---o-------+               |<----------+               |
|   (U)    |    |       |      (V)      |  Voucher  |      (W)      |
|          |    |       |               |  Response |               |
+----------+    |       +---------------+           +---------------+
              Voucher
~~~~~~~~~~~
{: #fig-overview title="Overview of the message flow. EDHOC is used on the constrained link between U and V. Voucher Info and Voucher are sent in EDHOC External Authorization Data (EAD). The link between V and W is not constrained." artwork-align="center"}



# Assumptions

The protocol is based on the following pre-existing relations between the device (U), the domain authenticator (V) and the enrollment server (W), see {{fig-trust}}.

* U and W have an explicit relation: U is configured with a public key of W, see {{device}}.
* V and W have an implicit relation, e.g., based on web PKI with trusted CA certificates, see {{domain-auth}}.
* U and V need not have any previous relation. This protocol establishes a relation between U and V.

Each of the three parties can gain protected communication with the other two during the protocol.

V may be able to access credentials over non-constrained networks, but U may be limited to constrained networks. Implementations wishing to leverage the zero-touch capabilities of this protocol are expected to support transmission of credentials from V to U by value during the EDHOC exchange, which will impact the message size depending on the type of credential used.

~~~~~~~~~~~ aasvg


         Explicit relation (e.g., from device manufacture)
     | <---------------------------------------------------> |
     |                                                       |
+----+-----+            +---------------+            +-------+-------+
|          |            |               |            |               |
|  Device  |    con-    |    Domain     |  not con-  |   Enrollment  |
|          |  strained  | Authenticator |  strained  |     Server    |
|   (U)    |  network   |      (V)      |  network   |      (W)      |
|          |            |               |            |               |
+----+-----+            +-------+-------+            +-------+-------+
     |                          |                            |
     | <----------------------> | <------------------------> |
          No previous relation        Implicit relation
                                    (e.g., web PKI based)
~~~~~~~~~~~
{: #fig-trust title="Overview of pre-existing relations." artwork-align="center"}



## Device (U) {#device}

To authenticate to V, the device (U) runs EDHOC in the role of Initiator with authentication credential CRED_U, for example, an X.509 certificate {{RFC5280}} or a CBOR Web Token (CWT, {{RFC8392}}). CRED_U may, for example, be carried by value in ID_CRED_I of EDHOC message_3 or be provisioned to V over a non-constrained network, leveraging a credential identifier in ID_CRED_I (see bottom of {{fig-protocol}}).

U also needs to identify itself to W, for which it reuses ID_CRED_I.
This means that the value used by U in ID_CRED_I needs to be unique enough to be understood by both V and W.
W will use ID_CRED_I to determine if the device with this identifier is authorized to enroll with V.

U is also provisioned with information about W:

* A static public DH key of W (G_W) used to establish secure communication with the enrollment server (see {{U-W}}).
* Location information about the enrollment server (LOC_W) that can be used by V to reach W. This is typically a URI but may alternatively be only the domain name.

## Domain Authenticator (V) {#domain-auth}

To authenticate to U, the domain authenticator (V) runs EDHOC in the role of Responder with an authentication credential CRED_V containing a public key of V, see {{V_2}}. This proves to U the possession of the private key corresponding to the public key of CRED_V. CRED_V typically needs to be transported to U in EDHOC (using  ID_CRED_R = CRED_V, see {{Section 3.5.2 of RFC9528}}) since there is no previous relation between U and V.

V and W need to establish a secure (confidentiality and integrity protected) connection for the Voucher Request/Response protocol.
Furthermore, W needs to access the same credential CRED_V that V uses with U (to compute the Voucher), and V needs to prove to W the possession of the private key corresponding to the public key of CRED_V.
It is RECOMMENDED that V authenticates to W using the same credential CRED_V as with U.

Note that the choice of protocols may affect which type of credential and methods should be used by V.
For example, in case V and W select TLS for the secure channel and PoP, then CRED_V is a X.509 certificate, and the EDHOC method used by V is signature-based.
Some of the possible combinations of protocols to secure the connection between V and W are listed in {{creds-table}} below.

| Secure channel between V and W | Proof-of-Possession from V to W | Type of CRED_V | EDHOC method used by V |
|--|--|--|--|
| [D]TLS 1.3 with mutual authentication, where V is the client and W is the server. | Provided by [D]TLS. | Restricted to types that are supported by both [D]TLS and EDHOC, e.g., X.509 certificates. | V MUST authenticate using a signature. |
| [D]TLS 1.3, where V is the client and W is the server. | Run an EDHOC session on top of the TLS-protected channel. | Any type supported by EDHOC, e.g., X.509, C509, CWT, or CCS. | Any method may be used. |
| EDHOC and OSCORE, where V is the initiator and W is the responder.    | Already provided by EDHOC during the setup of the secure channel. | Any type supported by EDHOC. | Any method may be used. |
{: #creds-table title="Examples of how to secure the connection between V and W." cols="l l l"}

Note also that the secure connection between V and W may be long-lived and reused for multiple voucher requests/responses.

Other details of proof-of-possession related to CRED_V and transport of CRED_V are out of scope of this document.



## Enrollment Server (W) {#authz-server}

The enrollment server (W) is assumed to have the private DH key corresponding to G_W, which is used to establish secure communication with the device (see {{U-W}}). W provides to U the authorization decision for enrollment with V in the form of a voucher (see {{voucher}}). Authorization policies are out of scope for this document.

Authentication credentials and communication security with V is described in {{domain-auth}}. To calculate the voucher, W needs access to message_1 and CRED_V as used in the EDHOC session between U and V, see {{voucher}}.

* W MUST verify that CRED_V is bound to the secure connection between W and V
* W MUST verify that V is in possession of the private key corresponding to the public key of CRED_V

W needs to be available during the execution of the protocol between U and V.

# The Protocol {#protocol}

## Overview {#protocol-overview}

The ELA protocol consist of three security sessions going on in parallel:

1. The EDHOC session between device (U) and (domain) authenticator (V)
2. Voucher Request/Response between authenticator (V) and enrollment server (W)
3. An exchange of voucher-related information, including the voucher itself, between device (U) and enrollment server (W), mediated by the authenticator (V).

{{fig-protocol}} provides an overview of the message flow detailed in this section. An outline of EDHOC is given in {{Section 2 of RFC9528}}.

~~~~~~~~~~~ aasvg

U                              V                                       W
|                              |                                       |
|                              |                                       |
|                              |        Establish secure channel       |
|                              +<---  ---  ---  ---  ---  ---  ---  -->|
|                              |      (e.g., TLS with server cert.)    |
|                              |                                       |
|                              |   Proof-of-possession w.r.t. CRED_V   |
|                              +<---  ---  ---  ---  ---  ---  ---  -->|
|                              |            (e.g., EDHOC)              |
|                              |                                       |
|                              |                                       |
|                              |                                       |

------------------------------------------------------------------------
                          CORE PROTOCOL

|                              |                                       |
|                              |                                       |
|         EDHOC message_1      |                                       |
+----------------------------->|                                       |
|                              |                                       |
|         EDHOC message_2      |                                       |
|<-----------------------------+                                       |
|                              |                                       |
|                              |                                       |
|         EDHOC message_3      |                                       |
+----------------------------->|                                       |
|         (EAD_3 = LOC_W)      |                                       |
|                              |                                       |
|                              |        Voucher Request (VREQ)         |
|                              +-------------------------------------->|
|                              |   (ID_CRED_I, SS, G_X, H_message_3)   |
|                              |                                       |
|                              |        Voucher Response (VRES)        |
|                              |<--------------------------------------+
|                              |          (Voucher, ?CRED_U)           |
|                              |                                       |
|         EDHOC message_4      |                                       |
|<-----------------------------+                                       |
|        (EAD_4 = Voucher)     |                                       |
|                              |                                       |

~~~~~~~~~~~
{: #fig-protocol title="Overview of ELA: W-assisted authorization of U and V to each other: EDHOC between U and V, and Voucher Request/Response between V and W. Before the protocol, V and W are assumed to have established a secure channel and performed proof-of-possession of relevant keys. Credential lookup of CRED_U may involve W or other credential database." artwork-align="center"}



## Reuse of EDHOC {#reuse}

The ELA protocol illustrated in {{fig-protocol}} reuses several components of EDHOC:

* G_X, the ephemeral public Diffie-Hellman key of U, is also used in the protocol between U and W.
In case U acts as Responder (see {{reverse-u-responder}}), G_Y is used instead.

* SUITES_I includes the cipher suite for EDHOC selected by U, and also defines the algorithms used between U and W (see {{Section 3.6 of RFC9528}}):

    * EDHOC AEAD algorithm: used to generate voucher
    * EDHOC hash algorithm: used for key derivation
    * EDHOC key exchange algorithm: used to calculate the shared secret between U and W

* EAD_3, EAD_4 are the External Authorization Data message fields of message_3 and message_4, respectively, see {{Section 3.8 of RFC9528}}.
In case U acts as Responder (see {{reverse-u-responder}}), EAD_2 and EAD_3 are used in message_2 and message_3, respectively.
This document specifies two new EAD items, with ead_label = TBD1 and TBD2, see {{iana-ead}}.

* ID_CRED_I and ID_CRED_R are used to identify the authentication credentials CRED_U and CRED_V, respectively. As shown at the bottom of {{fig-protocol}}, V may use W to obtain CRED_U.
By default, CRED_V is transported in ID_CRED_R in message_2, see {{V_2}}.
In case U is Responder, CRED_V is transported in ID_CRED_I in message_3.

The protocol also reuses the EDHOC_Extract and EDHOC_Expand key derivation from EDHOC (see {{Section 4 of RFC9528}}).

* The intermediate pseudo-random key PRK is derived using EDHOC_Extract():
    * PRK = EDHOC_Extract(salt, IKM)
         * where salt = 0x (the zero-length byte string)
         * IKM is computed as an ECDH cofactor Diffie-Hellman shared secret from the public key of W, G_W, and the private key corresponding to G_X (or v.v.), see Section 5.7.1.2 of {{NIST-800-56A}}.

The output keying material OKM is derived from PRK using EDHOC_Expand(), which is defined in terms of the EDHOC hash algorithm of the selected cipher suite SS, see {{Section 4.1.2 of RFC9528}}:

* OKM = EDHOC_Expand(PRK, info, length)

  where

~~~~~~~~~ cddl
info = (
   info_label : int,
   context : bstr,
   length : uint,
)
~~~~~~~~~~


## Device <-> Enrollment Server (U <-> W) {#U-W}

The protocol between U and W is carried between U and V in message_3 and message_4 ({{U-V}}), and between V and W in the Voucher Request/Response ({{V-W}}). The data is protected between the endpoints using secret keys derived from a shared secret (see {{reuse}}) as further detailed in this section.

### Voucher Info {#voucher_info}

The external authorization data EAD_1 contains a critical EAD item with ead_label = -TBD1 and ead_value = Voucher_Info, which is a CBOR byte string:

~~~~~~~~~~~ cddl
Voucher_Info = bstr .cborseq Voucher_Info_Seq

Voucher_Info_Seq = [ ; used as a CBOR sequence, not array
    LOC_W:      tstr,
]
~~~~~~~~~~~

where

* LOC_W is a text string used by V to locate W, e.g., a URI or a domain name.

### Voucher {#voucher}

The external authorization data EAD_4 contains an EAD item with ead_label = TBD2 and ead_value = Voucher.

The voucher is an assertion to U that W has authorized V.
It is encrypted using the EDHOC AEAD algorithm of the selected cipher suite SS specified in SUITE_I of EDHOC message_1.
It consists of the 'ciphertext' field of a COSE_Encrypt0 object, which is a byte string, as defined below.

~~~~~~~~~~~ cddl
Voucher = bstr
~~~~~~~~~~~

Its corresponding plaintext value consists of an opaque field that can be used by W to convey information to U, such as a voucher scope.
The authentication tag present in the ciphertext is also bound to message_3 and the credential of V as described below.

* The encryption key K_1 and nonce IV_1 are derived as specified below.
* 'protected' is a byte string of size 0
* 'plaintext' and 'external_aad' as below:

~~~~~~~~~~~ cddl
plaintext = (
    ?OPAQUE_INFO: bstr
)
~~~~~~~~~~~
~~~~~~~~~~~ cddl
external_aad = (
    H_handshake:  bstr,
    CRED_V:        bstr,
)
~~~~~~~~~~~

where

* OPAQUE_INFO is an opaque field provided by the application.
If present, it will contain application data that W may want to convey to U, e.g., a voucher scope.
Note that OPAQUE_INFO is opaque when viewed as an information element in EDHOC.
It is opaque to V, while the application in U and W can read its contents.

* H_handshake is the hash of EDHOC message_3, sent by V as part of the voucher request, see {{voucher_request}}.

* CRED_V is the credential used by V to authenticate to U and W, see {{V_2}} and {{creds-table}}.

The derivation of K_1 = EDHOC_Expand(PRK, info, length) uses the following input to the info struct (see {{reuse}}):

* info_label = 2
* context  = h'' (the empty CBOR string)
* length is length of the key of the EDHOC AEAD algorithm in bytes

The derivation of IV_1 = EDHOC_Expand(PRK, info, length) uses the following input to the info struct (see {{reuse}}):

* info_label = 3
* context = h''  (the empty CBOR string)
* length is length of the nonce of the EDHOC AEAD algorithm in bytes

## Device <-> Authenticator (U <-> V) {#U-V}

This section describes the processing in U and V, which includes the EDHOC protocol, see {{fig-protocol}}. Normal EDHOC processing is omitted here.

### Message 1 {#m1}

#### Processing in U

U composes EDHOC message_1 using authentication method, identifiers, etc. according to an agreed application profile, see {{Section 3.9 of RFC9528}}.
The selected cipher suite, in this document denoted SS, applies also to the interaction with W as detailed in {{reuse}}, in particular, with respect to the key agreement algorithm used between U and W.
As part of the normal EDHOC processing, U generates the ephemeral public key G_X that is reused in the interaction with W, see {{U-W}}.
U sends EDHOC message_1 to V.

#### Processing in V

V processes EDHOC message_1 as specified in {{RFC9528}}.

Note that as part of normal EDHOC processing, U and V may negotiate a selected cipher suite SS, as specified in {{Section 6.3.1 of RFC9528}}.

### Message 2 {#m2}

#### Processing in V  {#V_2}

V composes EDHOC message_2 as specified in {{Section 5.3.3 of RFC9528}}.

The type of CRED_V may depend on the selected mechanism for the establishment of a secure channel between V and W, See {{creds-table}}.

In case the network between U and V is constrained, it is recommended that CRED_V be a CWT Claims Set (CCS) {{RFC8392}}.
The CCS contains the public authentication key of V encoded as a COSE_Key in the 'cnf' claim, see {{Section 3.5.2 of RFC9528}}.
ID_CRED_R contains the CWT Claims Set with 'kccs' as COSE header_map, see {{Section 10.6 of RFC9528}}.

#### Processing in U

U receives EDHOC message_2 from V and processes it as specified in {{Section 5.3.3 of RFC9528}}.

### Message 3

#### Processing in U

U prepares EDHOC message_3 with EAD item (-TBD1, Voucher_Info) included in EAD_3, where Voucher_Info is specified in {{U-W}}.
The negative sign indicates that the EAD item is critical, see {{Section 3.8 of RFC9528}}.

EDHOC message_3 may be combined with an OSCORE-protected application request, see {{I-D.ietf-core-oscore-edhoc}}.

#### Processing in V

V receives EDHOC message_3 from U and processes it as specified in {{Section 5.4.3 of RFC9528}}, with the additional step of processing the EAD item in EAD_3.
Since the EAD item is critical, if V does not recognize it or it contains information that V cannot process, then V MUST abort the EDHOC session, see {{Section 3.8 of RFC9528}}.
Otherwise, the ead_label = TBD1 triggers the voucher request to W as described in {{V-W}}.
The exchange between V and W needs to be completed successfully for the EDHOC session to be continued.

As part of normal processing of EDHOC message_3, V must verify the credential of U.
It is up to the application to decide whether to verify the credential of U before or after the voucher request to W, see pre and post -verification processing of EAD items in {{I-D.ietf-lake-edhoc-impl-cons}}.

In case V has access to a credential database, it MAY query it with ID_CRED_I to obtain a corresponding CRED_U and verify the identity of U before making the Voucher request to W.
If the verification of CRED_U fails, the EDHOC session is aborted.
In this scenario, the EAD item in EAD_3 is processed as a post-verification item {{I-D.ietf-lake-edhoc-impl-cons}}.

Alternatively, V MAY proceed and perform a voucher request and wait for a voucher response.
If the response contains CRED_U, V then continues processing EDHOC message_3 where it verifies the credential of U.
Even upon reception of a valid Voucher from W, if the verification of CRED_U fails, the EDHOC session is aborted.
In this scenario, the EAD item in EAD_3 is processed as a pre-verification item {{I-D.ietf-lake-edhoc-impl-cons}}.

### Message 4 {#m4}

#### Processing in V  {#V_4}

At this point, V has authenticated U, and received a valid voucher response from W as described in {{V-W}}.

V sends EDHOC message_4 to U with the critical EAD item (-TBD2, Voucher) included in EAD_4, i.e., ead_label = TBD2 and ead_value = Voucher, as specified in {{voucher}}.

#### Processing in U

U receives EDHOC message_4 from V and processes it as specified in {{Section 5.3.3 of RFC9528}}, with the additional step of processing the EAD item in EAD_4.

If U does not recognize the EAD item or the EAD item contains information that U cannot process, then U MUST abort the EDHOC session, see {{Section 3.8 of RFC9528}}.
Otherwise, U MUST verify the Voucher using H_message_3, CRED_V, and the keys derived as in {{voucher}}. If the verification fails then U MUST abort the EDHOC session.

If OPAQUE_INFO is present, it is made available to the application.

## Authenticator <-> Enrollment Server (V <-> W) {#V-W}

It is assumed that V and W have set up a secure connection, W has accessed the authentication credential CRED_V to be used in the EDHOC session between V and U, and that W has verified that V is in possession of the private key corresponding to CRED_V, see {{domain-auth}} and {{authz-server}}.
V and W run the Voucher Request/Response protocol over the secure connection.

### Voucher Request {#voucher_request}

#### Processing in V

V sends the voucher request to W.
The Voucher Request SHALL be a CBOR array as defined below:

~~~~~~~~~~~ cddl
Voucher_Request = [
    ID_CRED_I:    bstr,
    SS:             int,
    G_U:            bstr,
    H_handshake:    bstr,
]
~~~~~~~~~~~

where

* ID_CRED_I is the identifier of U extracted from message_3, from which CRED_U can be obtained
* SS is the selected cipher suite used in the EDHOC session between U and V
* G_U is the ephemeral public key (G_X) of U
* H_handshake is the hash of message_3 (H_message_3). It is computed using the EDHOC hash algorithm of the selected cipher suite SS specified in SUITE_I of EDHOC message_1.

#### Processing in W

W receives and parses the voucher request received over the secure connection with V.
W extracts from Voucher_Request:

* ID_CRED_I - an optional identifier for CRED_U.
* SS - the selected cipher suite.
* G_U - the ephemeral public key of U.
* H_handshake - the hash of message_3.

W uses H_handshake as a session identifier, and associates it to the device identifier ID_CRED_I.
Note that G_U is a unique ephemeral key, therefore H_handshake is expected to be unique.

If processing fails up until this point, the protocol SHALL be aborted with an error code signaling a generic issue with the request, see {{rest-voucher-request}}.

W uses ID_CRED_I to look up the associated authorization policies for U and enforces them. This is out of scope for the specification.

If ID_CRED_I is known by W, but authorization fails, the protocol SHALL be aborted with an error code signaling an access control issue, see {{err-handling}} and {{rest-voucher-request}}.

W MAY use ID_CRED_I to retrieve the corresponding CRED_U to be sent in the voucher response, see {{voucher_response}}.

### Voucher Response {#voucher_response}

#### Processing in W

In case a Voucher is needed (as determined by the application), W retrieves CRED_V associated with the secure connection with V, and constructs the Voucher for the device with identifier ID_CRED_I (see {{voucher}}).

W generates the voucher response and sends it to V over the secure connection.
The Voucher_Response SHALL be a CBOR array as defined below:

~~~~~~~~~~~ cddl
Voucher_Response = [
    Voucher:        bstr,
    ? CRED_U:       bstr,
]
~~~~~~~~~~~

where

* The Voucher is defined in {{voucher}}.
* CRED_U is an optional field corresponding to the credential of U, extracted according to the ID_CRED_I field passed in the voucher request.

W signals the successful processing of Voucher_Request via a status code in the REST interface, as defined in {{rest-voucher-request}}.

#### Processing in V

V receives the voucher response from W over the secure connection.
If the voucher response is successfully received from W, then V responds to U with EDHOC message_4 as described in {{V_4}}.

## Error Handling {#err-handling}
This section specifies a new EDHOC error code and how it is used in ELA.

### EDHOC Error "Access denied"

This section specifies the new EDHOC error "Access denied", see {{fig-error-codes}}.

~~~~~~~~~~~ aasvg
+----------+----------------+----------------------------------------+
| ERR_CODE | ERR_INFO Type  | Description                            |
+==========+================+========================================+
|     TBD3 | error_content  | Access denied                          |
+----------+----------------+----------------------------------------+
~~~~~~~~~~~
{: #fig-error-codes title="EDHOC error code and error information for ‘Access denied’."}

Error code TBD3 is used to indicate to the receiver that access control has been applied and the sender has aborted the EDHOC session.
The ERR_INFO field contains error_content which is a CBOR Sequence consisting of an integer and an optional byte string.

~~~~~~~~~~~ cddl
error_content = (
  REJECT_TYPE : int,
  ? REJECT_INFO : bstr,
)
~~~~~~~~~~~

The purpose of REJECT_INFO is for the sender to provide verifiable and actionable information to the receiver about the error, so that an automated action may be taken to enable access.

~~~~~~~~~~~ aasvg
+-------------+---------------+--------------------------------------+
| REJECT_TYPE | REJECT_INFO   | Description                          |
+=============+===============+======================================+
|           0 | -             | No REJECT_INFO                       |
+-------------+---------------+--------------------------------------+
|           1 | bstr          | REJECT_INFO from trusted third party |
+-------------+---------------+--------------------------------------+
~~~~~~~~~~~
{: #fig-reject title="REJECT_TYPE and REJECT_INFO for ‘Access denied’."}

### Error handling in W, V, and U

ELA uses the EDHOC Error "Access denied" in the following way:

* W generates error_content and transfers it to V via the secure connection.
  If REJECT_TYPE is 1, then REJECT_INFO is encrypted from W to U using the EDHOC AEAD algorithm.
  W signals the error via an appropriate status code in the REST interface, as defined in {{rest-voucher-request}}.
* V receives error_content, prepares an EDHOC "Access denied" error, and sends it to U.
* U receives the error message and extracts the error_content.
  If REJECT_TYPE is 1, then U decrypts REJECT_INFO, based on which it may retry to gain access.

The encryption of REJECT_INFO follows a procedure analogous to the one defined in {{voucher}}, with the following differences:

~~~~~~~~~~~ cddl
plaintext = (
    OPAQUE_INFO:     bstr,
 )
~~~~~~~~~~~
~~~~~~~~~~~ cddl
external_aad = (
    H_handshake:    bstr,
 )
~~~~~~~~~~~

where

* OPAQUE_INFO is an opaque field that contains actionable information about the error.
  It may contain, for example, a list of suggested Vs through which U should join instead.

* H_handshake is the hash of EDHOC message_3, calculated from the associated voucher request, see {{voucher_request}}.

## Reverse flow with U as Responder {#reverse-u-responder}

TODO(fix this section with new ID_CRED_I-based solution)

This section presents a protocol variant in which U is the EDHOC Responder.
This may allow optimizations in certain constrained network technologies.
For example, one use case is having V broadcast message_1, to which U responds with a message_2 whose EAD_2 field contains Voucher_Info.

Note that this is different from the EDHOC reverse message flow defined in {{Appendix A.2.2 of RFC9528}}, since we make no assumption about whether U or V is a CoAP server.

### U is the Initiator {#u-initiator}

For clarity, we first present the "default flow" with U as Initiator, as described in {{protocol-overview}} and {{U-V}}.
Note that Voucher_Info and Voucher are carried in EDHOC message_1 and message_2, respectively.

~~~~~~~~~~~ aasvg
+-----+-----+                   +-----------+
|     U     |                   |     V     |
| Initiator |                   | Responder |
+-----+-----+                   +-----+-----+
      |                               |
      |                               |
      |       EDHOC message_1         |                              W
      +------------------------------>|                              |
      | EAD_1 = (-TBD1, Voucher_info) |                              |
      |                               |             VREQ             |
      |                               +----------------------------->|
      |                               | (SS, G_X, Voucher_Info,      |
      |                               |  H_message_1, ?opaque_state) |
      |                               |                              |
      |                               |             VRES             |
      |                               +<---------------------------->|
      |                               |    (?Voucher, ?opaque_state) |
      |       EDHOC message_2         |
      +<------------------------------|
      |   EAD_2 = (-TBD2, ?Voucher)   |
      |                               |
      |       EDHOC message_3         |
      +------------------------------>|
      |                               |
      |                               |
~~~~~~~~~~~
{: #fig-u-initiator title="In ELA default flow, U is the EDHOC Initiator." artwork-align="center"}

In the ELA default flow, once message_2 processing is finalized (including processing of EAD_2), U considers V authenticated through W.

### U is the Responder {#u-responder}

ELA also works with U as the EDHOC Responder, a setup we refer to as the "ELA reverse flow", as shown in {{fig-u-responder}}.

We present this variant as a set of changes to the regular protocol flow.
That is, here we only describe the differences in processing, when compared to the ELA default flow.

Here is a summary of the changes needed in the ELA reverse flow:

* Voucher_Info and Voucher are transported in EDHOC message_2 and message_3, respectively (instead of message_1 and message_2).
* The EAD_2 and EAD_3 fields carry critical EAD items identified with labels -TBD1 and -TBD2, respectively.
* The VREQ / VRES protocol takes place between message_2 and message_3.
* The Voucher_Request carries G_Y instead of G_X, and the transcript hash TH_2 instead of the hash H_message_1.

~~~~~~~~~~~ aasvg
+-----+-----+                   +-----------+
|     U     |                   |     V     |
| Initiator |                   | Responder |
+-----+-----+                   +-----+-----+
      |                               |
      |       Trigger Message         |
      +- - - - - - - - - - - - - - - >|
      |                               |
      |       EDHOC message_1         |
      +<------------------------------|
      |                               |
      |                               |
      |       EDHOC message_2         |                              W
      +------------------------------>|                              |
      | EAD_2 = (-TBD1, Voucher_info) |                              |
      |                               |             VREQ             |
      |                               +----------------------------->|
      |                               | (SS, G_Y, Voucher_Info,      |
      |                               |  TH_2, ?opaque_state)        |
      |                               |                              |
      |                               |             VRES             |
      |                               +<---------------------------->|
      |                               |    (Voucher, ?opaque_state)  |
      |     EDHOC message_3           |
      +<------------------------------|
      |   EAD_3 = (-TBD2, ?Voucher)   |
      |                               |
      |                               |
~~~~~~~~~~~
{: #fig-u-responder title="ELA when U is the EDHOC Responder." artwork-align="center"}

The following detail how the processing changes in each of the three security sessions.

The way to interpret the subsections below is as follows.
The ELA reverse flow described in this section uses most of the ELA default flow processing ({{protocol-overview}} to {{err-handling}}), except by the changes detailed in {{reverse-u-w}},  {{reverse-u-v}}, and  {{reverse-v-w}}.

#### Reverse U <-> W {#reverse-u-w}

The protocol between U and W is carried between U and V in message_2 and message_3, and between V and W in the Voucher Request/Response ({{V-W}}).

Voucher Info:

* The EAD_2 item has ead_label = -TBD1 and ead_value = Voucher_Info.

Voucher:

* H_handshake is the transcript hash TH_2, sent by V as part of the voucher request, see {{reverse-v-w}}.

#### Reverse U <-> V {#reverse-u-v}

Message 1:

* V composes message_1 and sends it to U.
* U processes message_1 and extracts SS.

Message 2:

* U composes message_2 and generates G_Y, which is reused in the interaction with W.
* U sends message_2 with critical EAD item (-TBD1, Voucher_Info) included in EAD_2.
* V processes message_2 and the EAD item in EAD_2, extracting the Voucher_Info struct.
* V sends the voucher request to W.

Message 3:

* V receives the voucher response from W.
* V sends message_3 with critical EAD item (-TBD2, ?Voucher) included in EAD_3.
* Y processes message_3 and the EAD item in EAD_3.

#### Reverse V <-> W {#reverse-v-w}

Processing in V:

* The Voucher_Request fields are prepared as defined in {{voucher_request}}, with the following changes:
  * G_U is set to G_Y, which is the ephemeral public key of U as extracted from message_2.
  * Voucher_Info is as extracted from the EAD_2 field of message_2.
  * H_handshake is the transcript hash TH_2, computed by V as specified in {{Section 5.3.2 of RFC9528}}.

Processing in W happens as specified in {{voucher_request}}.

### Interoperability considerations

A Device (U) MUST implement one of the ELA flows, and it MAY choose to implement both.

V MUST support the regular flow and MAY support the reverse flow.

From the point of view of W, there is no difference whether U and V run as EDHOC Initiator or Responder.

### Security implications

When using the reverse flow, U shares its identity before it can learn (1) V's identity and (2) whether or not the Voucher is valid.

In the reverse flow, Voucher_Info is confidentiality and integrity protected, while Voucher is also authenticated.
These properties are inherited from EDHOC message_2 and message_3.
This is a higher level of protection than with the regular flow.

# REST Interface at W {#rest_interface}

The interaction between V and W is enabled through a RESTful interface exposed by W.
This RESTful interface MAY be implemented using either HTTP or CoAP.
V SHOULD access the resources exposed by W through the protocol indicated by the scheme in the LOC_W URI.

## Scheme "https" {#scheme-https}
In case the scheme indicates "https", V MUST perform a TLS handshake with W and access the resources defined in {{uris}} using HTTP.
If the authentication credential CRED_V can be used in a TLS handshake, e.g., an X.509 certificate of a signature public key, then V SHOULD use it to authenticate to W as a client.
If the authentication credential CRED_V cannot be used in a TLS handshake, e.g., if the public key is a static Diffie-Hellman key, then V SHOULD first perform a TLS handshake with W using available compatible keys.
V MUST then perform an EDHOC session over the TLS connection proving to W the possession of the private key corresponding to CRED_V.
Performing the EDHOC session is only necessary if V did not authenticate with CRED_V in the TLS handshake with W.

The relationship between V and W is long-lived.  HTTP/1.1 and higher support persistent connections, and SHOULD be used in order to reduce overhead if a flood of new devices need to be onboarded.  Support for TLS session resumptions tickets {{RFC8446, Section 2.2}} is appropriate for longer term associations.
While a policy for renewal of the TLS connection should be applied, it is out of scope of this document.

## Scheme "coaps"
In case the scheme indicates "coaps", V SHOULD perform a DTLS handshake with W and access the resources defined in {{uris}} using CoAP.
The normative requirements in {{scheme-https}} on performing the DTLS handshake and EDHOC session remain the same, except that TLS is replaced with DTLS.
As in {{scheme-https}}, it is RECOMMENDED to allow reuse of the DTLS session.

## Scheme "coap"
In case the scheme indicates "coap", V SHOULD perform an EDHOC session with W, as specified in {{Appendix A of RFC9528}} and access the resources defined in {{uris}} using OSCORE and CoAP.
The authentication credential in this EDHOC session MUST be CRED_V.
As in {{scheme-https}}, it is RECOMMENDED to allow reuse of the EDHOC session.

## URIs {#uris}

The URIs defined below are valid for both HTTP and CoAP.
W MUST support the use of the path-prefix "/.well-known/", as defined in {{RFC8615}}, and the registered name "lake-authz".
A valid URI in case of HTTP thus begins with

* "https://www.example.com/.well-known/lake-authz"

In case of CoAP with DTLS:

* "coaps://example.com/.well-known/lake-authz"

In case of EDHOC and OSCORE:

* "coap://example.com/.well-known/lake-authz"

Each operation specified in the following is indicated by a path-suffix.

### Voucher Request (/voucherrequest) {#rest-voucher-request}

To request a voucher, V MUST issue a request such that:

* Method is POST
* Payload is the serialization of the Voucher Request object, as specified in {{voucher_request}}.
* Content-Format (Content-Type) is set to "application/lake-authz-voucherrequest+cbor"

In case of successful processing at W, W MUST issue a response such that:

* Status code is 200 OK if using HTTP, or 2.04 Changed if using CoAP
* Payload is the serialized Voucher Response object, as specified in {{voucher_response}}
* Content-Format (Content-Type) is set to "application/lake-authz-voucherresponse+cbor"

In case of error, two cases should be considered:

* U cannot be identified: this happens either if W fails to process the Voucher Request, or if it succeeds but ID_CRED_I is considered unknown to W. In this case, W MUST reply with 400 Bad Request if using HTTP, or 4.00 if using CoAP.
* U is identified but unauthorized: this happens if W is able to process the Voucher Request, and W recognizes ID_CRED_I as a known device, but the access policies forbid enrollment. For example, the policy could enforce enrollment within a delimited time window, via a specific V, etc. In this case, W MUST reply with a 403 Forbidden code if using HTTP, or 4.03 if using CoAP; the payload is the serialized error_content object, with Content-Format (Content-Type) set to "application/lake-authz-vouchererror+cbor". The payload MAY be used by V to prepare an EDHOC error "Access Denied", see {{err-handling}}.

### Certificate Request (/certrequest)

V requests the public key certificate of U from W through the "/certrequest" path-suffix.
To request U's authentication credential, V MUST issue a request such that:

* Method is POST
* Payload is the serialization of the ID_CRED_I object, as received in EDHOC message_3.
* Content-Format (Content-Type) is set to "application/lake-authz-certrequest+cbor"

In case of a successful lookup of the authentication credential at W, W MUST issue a response such that:

* Status code is 200 OK if using HTTP, or 2.04 Changed if using CoAP
* Payload is the serialized CRED_U
* Content-Format (Content-Type) is set to "application/lake-authz-certresponse+cbor"

# Security Considerations  {#sec-cons}

This specification builds on and reuses many of the security constructions of EDHOC, e.g., shared secret calculation and key derivation. The security considerations of EDHOC {{RFC9528}} apply with modifications discussed here.

EDHOC provides identity protection of the Initiator, here the device.
In ELA, the device U will share its identity with an authenticated V, albeit before knowing (via the Voucher received from W) whether U is authorized to interact with W.

Although W learns about the identity of U after receiving VREQ, this information must not be disclosed to V, until U has revealed its identity to V with ID_CRED_I in message_3.
W may be used for lookup of CRED_U from ID_CRED_I, or this credential lookup function may be separate from the authorization function of W, see {{fig-protocol}}.
FIXME(trust model changes, since now ID_CRED_I is always revealed to V).
The trust model used here is that U decides to which V it reveals its identity.
In an alternative trust model where U trusts W to decide to which V it reveals U's identity, CRED_U could be sent in Voucher Response.

 As noted in {{Section 9.2 of RFC9528}} an ephemeral key may be used to calculate several ECDH shared secrets. In this specification, the ephemeral key G_X is also used to calculate G_XW, the shared secret with the enrollment server.

FIXME(just remove this paragraph?).
The private ephemeral key is thus used in the device for calculations of key material relating to both the authenticator and the enrollment server. There are different options for where to implement these calculations. One option is as an addition to EDHOC, i.e., to extend the EDHOC API in the device, so that EDHOC can import the public key of W (G_W) and the device identifier of U (ID_U), and then produce the encryption of ID_U which is included in Voucher_Info in EAD_1.

# IANA Considerations  {#iana}

## EDHOC External Authorization Data Registry {#iana-ead}

IANA has registered the following entries in the "EDHOC External Authorization Data" registry under the group name "Ephemeral Diffie-
   Hellman Over COSE (EDHOC)".

| Label | Value Type | Description |
| TBD1 | bstr | Voucher_Info structure, prepared by the Device (U). |
| TBD2 | bstr | Voucher structure, prepared by the Enrollment Server (W). |
{: #ead-table title="Addition to the EDHOC EAD registry" cols="r l l"}

The ead_label = TBD1 corresponds to the ead_value = Voucher_Info, which can be carried in either EAD_1 or EAD_2, depending on whether U acts as EDHOC Initiator or Responder, see {{reverse-u-responder}}.

The ead_label = TBD2 corresponds to ead_value = Voucher, and can be carried in either EAD_2 or EAD_3, see {{reverse-u-responder}}.

Note for IANA reviewers: the preferred value range is 0-23 (Standards Action with Expert Review).

## The Well-Known URI Registry

IANA has registered the following entry in "The Well-Known URI Registry", using the template from {{RFC8615}}:

* URI suffix: lake-authz
* Change controller: IETF
* Specification document: \[\[this document\]\]
* Status: permanent
* Related information: None

## Well-Known Name Under ".arpa" Name Space

This document allocates a well-known name under the .arpa name space according to the rules given in {{RFC3172}} and {{RFC6761}}.
The name "lake-authz.arpa" is requested.
No subdomains are expected, and addition of any such subdomains requires the publication of an IETF Standards Track RFC.
No A, AAAA, or PTR record is requested.

### Domain Name Reservation Considerations

As required by {{RFC6761}}, the following considerations apply to the reservation of "lake-authz.arpa":

1. Users:
Are human users expected to recognize these names as special and use them differently? In what way?

No. This name is not intended for direct use or recognition by human users.

2. Application Software:
Are writers of application software expected to make their software recognize these names as special and treat them differently? In what way? (For example, if a human user enters such a name, should the application software reject it with an error message?)

Yes. Applications that implement ELA and use CoAP may include "lake-authz.arpa" in the URI-Host option when the Device (U) does not yet know the address or identity of the Authenticator (V), such as during zero-touch enrollment.

3. Name Resolution APIs and Libraries:
Are writers of name resolution APIs and libraries expected to make their software recognize these names as special and treat them differently? If so, how?

No.

4. Caching DNS Servers:
Are developers of caching domain name servers expected to make their implementations recognize these names as special and treat them differently? If so, how?

No.

5. Authoritative DNS Servers:
Are developers of authoritative domain name servers expected to make their implementations recognize these names as special and treat them differently? If so, how?

No.

6. DNS Server Operators:
Does this reserved Special-Use Domain Name have any potential impact on DNS server operators? If they try to configure their authoritative DNS server as authoritative for this reserved name, will compliant name server software reject it as invalid? Do DNS server operators need to know about that and understand why? Even if the name server software doesn't prevent them from using this reserved name, are there other ways that it may not work as expected, of which the DNS server operator should be aware?

No.

7. DNS Registries/Registrars:
How should DNS Registries/Registrars treat requests to register this reserved domain name? Should such requests be denied? Should such requests be allowed, but only to a specially designated entity? (For example, the name "www.example.org" is reserved for documentation examples and is not available for registration; however, the name is in fact registered; and there is even a website at that name, which states circularly that the name is reserved for use in documentation and cannot be registered!)

Any requests to register this domain name should be denied.


## Media Types Registry

IANA has added the media types "application/lake-authz-voucherrequest+cbor" to the "Media Types" registry.

### application/lake-authz-voucherrequest+cbor Media Type Registration

* Type name: application
* Subtype name: lake-authz-voucherrequest+cbor
* Required parameters: N/A
* Optional parameters: N/A
* Encoding considerations: binary (CBOR)
* Security considerations: See {{sec-cons}} of this document.
* Interoperability considerations: N/A
* Published specification: [[this document]] (this document)
* Application that use this media type: To be identified
* Fragment identifier considerations: N/A
* Additional information:
    * Magic number(s): N/A
    * File extension(s): N/A
    * Macintosh file type code(s): N/A
* Person & email address to contact for further information: IETF LAKE Working Group (lake@ietf.org)
* Intended usage: COMMON
* Restrictions on usage: N/A
* Author: LAKE WG
* Change Controller: IETF

## CoAP Content-Formats Registry

IANA has added the following Content-Format number in the "CoAP Content-Formats" registry under the registry group "Constrained RESTful Environments (CoRE) Parameters".

| Content Type | Content Encoding | ID | Reference |
| application/lake-authz-voucherrequest+cbor | - | TBD3 | [[this document]] |
{: #coap-content-formats title="Addition to the CoAP Content-Formats registry" cols="l l l"}

Note for IANA reviewers: the preferred value range is 0-255 (Expert Review).

--- back

# Optimization Strategies {#optimization-strat}

When ELA is used for zero-touch enrollment of IoT devices, U may have little to no knowledge about V's available in its vicinity.
This may lead to situations where U retries several times at different V's until it finds one that works.
This section presents two optimization strategies for such cases.
They were developed to address scenarios where V's are radio gateways to which U wants to enroll, but may also be applicable to other use cases.

## U broadcasts message_1 {#strat-anycast}

This strategy consists in U broadcasting EDHOC message_1.
When each of the V's in radio range of U receive message_1, one of the following can happen:

- V does not implement EDHOC, and drops the message
- V does not implement ELA, and drops the message (even though the EAD_1 option is critical, broadcast messages should not have error replies)
- V forwards message_1 to W as VREQ, but W does not authorize it, and error handling is applied
- V forwards message_1 to W as VREQ, W authorizes it, and the protocol continues normally

U is expected to receive and process at most one message_2 as response, which contains the Voucher.
In case U receives additional message_2's, they MUST be silently dropped.

This strategy may increase the number of messages that need to be processed by V and W, in exchange for reducing resource usage in U.

Security concerns related to this strategy, including potential reuse of G_X and double processing of message_2, are discussed in {{sec-cons}}.

## V advertises support for ELA {#strat-advertise}

In this strategy, V shares some information (V_INFO) with a potential U, that can help it decide whether to try to enroll with that V.

The exact contents of the V_INFO structure, as well as the mechanism used to transport it, will depend on the underlying communication technology and also on application needs.
For example, V_INFO may state that:

- V implements ELA -- similarly to how EAPOL {{IEEE802.1X}} frames state support for IEEE 802.1X.

- V is part of a certain domain -- similarly to how Eduroam {{RFC7593}} is used in the SSID field of IEEE 802.11 packets

V_INFO can be sent over a network beacon (see {{adv-beacon}}), which may require technology specific profiling, e.g., the IEEE 802.15.4 enhanced beacon may be extended according to {{RFC8137}}.
Alternatively, V_INFO can be sent as part of an EAD field, as shown in {{adv-ead1}}.

As a guideline for implementers, we define the following field that can be included in a V_INFO structure:

~~~~~~~~~~~ cddl
DOMAIN_ID: bstr
~~~~~~~~~~~

The DOMAIN_ID field identifies the domain to which V belongs to, for example an URL or UUID.

Below are three examples of how the advertisement strategy may be applied according to different application needs.
The examples include sending V_INFO in network beacons, as part of EAD_1 in reverse message flow, or as part of a periodic CoAP multicast packet.
The advantages, costs, and security impacts of each approach are also discussed.

### V_INFO in network beacons {#adv-beacon}

This approach allows carrying V_INFO in beacons sent over the network layer, as shown in {{fig-adv-beacon}}.
It requires that the network layer offers a mechanism to configure its beacon packets.
Depending on the network type, a solicitation packet may also be needed, as is the case of non-beaconed IEEE 802.15.4 and BLE with GATT.

~~~~~~~~~~~ aasvg
+-------+--------+                       +-------+--------+
| Init  | Client |                       | Resp  | Server |
+-------+--------+                       +----------------+
|       U        |                       |       V        |
+----------------+                       +----------------+
        |                                        |
        + - - - - - - - - - - - - - - - - - - -->|
        |     Optional network solicitation      |
        |                                        |
        |<---------------------------------------+
        |   Network discovery (contains V_INFO)  |
        |                                        |
        |            EDHOC message_1             |
        +--------------------------------------->|
        |        (?EAD_1 = Voucher_Info)         |
        |                                        |

         ( ... protocol continues normally ... )
~~~~~~~~~~~
{: #fig-adv-beacon title="Advertising ELA using V_INFO in network-layer beacons." artwork-align="center"}

This strategy can be used, for example, in IEEE 802.15.4, where an Enhanced Beacon {{IEEE802.15.4}} can be used to transmit V_INFO.
Specifically, a new information element for carrying V_INFO can be defined according to {{RFC8137}}.

This approach has the advantage of requiring minimal changes to the default protocol as presented in {{protocol-overview}}, i.e., no reverse flow.
It requires, however, some profiling of the lower layer beacons.

### V_INFO in EAD_1 {#adv-ead1}

The ELA reverse flow (see {{reverse-u-responder}}) allows implementing advertising where U first sends a trigger packet, in the format of a CoAP request that is broadcasted to the newtork.
When a suitable V receives the solicitation, if it implements ELA, it should respond with an EDHOC message_1 whose EAD_1 has label -TBD1 and value V_INFO (see Section {{optimization-strat}}).

~~~~~~~~~~~ aasvg
+-------+--------+                +-------+--------+
| Resp  | Client |                | Init  | Server |
+-------+--------+                +----------------+
|       U        |                |       V        |
+----------------+                +----------------+
        |                                 |
        +-------------------------------->|
        |   CoAP discovery/solicitation   |
        |                                 |
        |        EDHOC message_1          |
        +<--------------------------------|
        |       (?EAD_1 = V_INFO)         |
        |                                 |

     ( ... reverse flow continues normally ... )
~~~~~~~~~~~
{: #fig-adv-ead1 title="Advertising ELA using V_INFO in EAD_1, eploying the EDHOC reverse flow with U as responder." artwork-align="center"}

Note that V will only reply if it supports ELA.
V_INFO can be structured to contain only an optional domain identifier:

~~~ cddl
V_INFO = (
  ?DOMAIN_ID: bstr,
)
~~~

This approach enables a simple filtering mechanism, where only V's that support ELA will reply.
It also encrypts Voucher_Info (as part of EAD_2), whereas it is sent in the clear in the original flow.
In addition, it may not require layer-two profiling (in case the network allows transporting data before authorization).
Finally, note that the reverse flow with U as Responder protects the identity of V (instead of U's as in the forward flow).

### V_INFO in a CoAP Multicast Packet {#adv-coap-mult}

In this approach, V periodically multicasts a CoAP packet containing V_INFO, see {{fig-adv-coap-mult}}.
Upon receiving one or more CoAP messages and processing V_INFO, U can decide whether or not to initiate the ELA protocol with a given V.
Next, the application can either keep U acting as a server, and thus employ the EDHOC reverse flow, or implement a CoAP client and use the forward flow.

~~~~~~~~~~~ aasvg
+--------+---------+                       +--------+---------+
|  Init  | Cli/Ser |                       |  Resp  | Cli/Ser |
+--------+---------+                       +------------------+
|        U         |                       |        V         |
+------------------+                       +------------------+
         |                                          |
         |          POST /ela-advertisement         |
         |<-----------------------------------------+
         |     CoAP multicast (contains V_INFO)     |
         |                                          |
         |              EDHOC message_1             |
         +----------------------------------------->|
         |          (?EAD_1 = Voucher_Info)         |
         |                                          |

          ( ... protocol continues normally ... )
~~~~~~~~~~~
{: #fig-adv-coap-mult title="Advertising ELA using the network layer." artwork-align="center"}

The V_INFO structure is sent as part of the CoAP payload.
It is encoded as a CBOR sequence:

~~~ cddl
V_INFO = (
  ?DOMAIN_ID: bstr,
)
~~~

One advantage of this approach is that, since U is the initiator, its identity is protected in the context of the EDHOC handshake.
On the other hand, the periodic multicast may have resource usage impacts in the network.

# Use with Constrained Join Protocol (CoJP)

This section outlines how ELA is used for network enrollment and parameter provisioning.
An IEEE 802.15.4 network is used as an example of how a new device (U) can be enrolled into the domain managed by the domain authenticator (V).

~~~~~~~~~~~ aasvg
U                                    V                              W
|                                    |                              |
|                                    |                              |
+ - - - - - - - - - - - - - - - - -->|                              |
|    Optional network solicitation   |                              |
|<-----------------------------------+                              |
|          Network discovery         |                              |
|                                    |                              |
+----------------------------------->|                              |
|          EDHOC message_1           |                              |
|                                    +----------------------------->|
|                                    |    Voucher Request (VREQ)    |
|                                    |<-----------------------------+
|                                    |    Voucher Response (VRES)   |
|<-----------------------------------+                              |
|          EDHOC message_2           |                              |
|                                    |                              |
|                                    |                              |
+----------------------------------->|                              |
|   EDHOC message_3 + CoJP request   |                              |
|                                    |                              |
+<-----------------------------------|                              |
|            CoJP response           |                              |
|
~~~~~~~~~~~
{: #fig-cojp title="Use of draft-ietf-lake-authz with CoJP." artwork-align="center"}


## Network Discovery

When a device first boots, it needs to discover the network it attempts to join.
The network discovery procedure is defined by the link-layer technology in use.
In case of Time-slotted Channel Hopping (TSCH) networks, a mode of {{IEEE802.15.4}}, the device scans the radio channels for Enhanced Beacon (EB) frames, a procedure known as passive scan.
EBs carry the information about the network, and particularly the network identifier.
Based on the EB, the network identifier, the information pre-configured into the device, the device makes the decision on whether it should join the network advertised by the received EB frame.
This process is described in {{Section 4.1 of RFC9031}}.
In case of other, non-TSCH modes of IEEE 802.15.4, it is possible to use the active scan procedure and send solicitation frames.
These solicitation frames trigger the nearest network coordinator to respond by emitting a beacon frame.
The network coordinator emitting beacons may be multiple link-layer hops away from the domain authenticator (V), in which case it plays the role of a Join Proxy (see {{RFC9031}}).
The Join Proxy does not participate in the protocol and acts as a transparent router between the device and the domain authenticator.
For simplicity, {{fig-cojp}} illustrates the case when the device and the domain authenticator are a single hop away and can communicate directly.

## The Enrollment Protocol with Parameter Provisioning

### Flight 1

Once the device has discovered the network it wants to join, it constructs EDHOC message_1, as described in {{U-V}}.
The device SHALL map the message to a CoAP request:

* The request method is POST.
* The type is Confirmable (CON).
* The Proxy-Scheme option is set to "coap".
* The Uri-Host option is set to "lake-authz.arpa". This is an anycast type of identifier of the domain authenticator (V) that is resolved to its IPv6 address by the Join Proxy.
* By means of Uri-Path options, the Uri-Path is set to ".well-known/edhoc".
* The payload is the (true, EDHOC message_1) CBOR sequence, where EDHOC message_1 is constructed as defined in {{U-V}}.

### Flight 2

The domain authenticator receives message_1 and processes it as described in {{U-V}}.
The message triggers the exchange with the enrollment server, as described in {{V-W}}.
If the exchange between V and W completes successfully, the domain authenticator prepares EDHOC message_2, as described in {{U-V}}.
The authenticator SHALL map the message to a CoAP response:

* The response code is 2.04 Changed.
* The payload is the EDHOC message_2, as defined in {{U-V}}.

### Flight 3

The device receives EDHOC message_2 and processes it as described in {{U-V}}.
Upon successful processing of message_2, the device prepares flight 3, which is an OSCORE-protected CoJP request containing an EDHOC message_3, as described in {{I-D.ietf-core-oscore-edhoc}}.
EDHOC message_3 is prepared as described in {{U-V}}.
The OSCORE-protected payload is the CoJP Join Request object specified in {{Section 8.4.1 of RFC9031}}.
OSCORE protection leverages the OSCORE Security Context derived from the EDHOC session, as specified in Appendix A of {{RFC9528}}.
To that end, {{I-D.ietf-core-oscore-edhoc}} specifies that the Sender ID of the client (device) must be set to the connection identifier selected by the domain authenticator, C_R.
OSCORE includes the Sender ID as the kid in the OSCORE option.
The network identifier in the CoJP Join Request object is set to the network identifier obtained from the network discovery phase.
In case of IEEE 802.15.4 networks, this is the PAN ID.

The device SHALL map the message to a CoAP request:

* The request method is POST.
* The type is Confirmable (CON).
* The Proxy-Scheme option is set to "coap".
* The Uri-Host option is set to "lake-authz.arpa".
* The Uri-Path option is set to ".well-known/edhoc".
* The EDHOC option {{I-D.ietf-core-oscore-edhoc}} is set and is empty.
* The payload is prepared as described in {{Section 3.2 of I-D.ietf-core-oscore-edhoc}}, with EDHOC message_3 and the CoJP Join Request object as the OSCORE-protected payload.

Note that the OSCORE Sender IDs are derived from the connection identifiers of the EDHOC session.
This is in contrast with {{RFC9031}} where ID Context of the OSCORE Security Context is set to the device identifier (pledge identifier).
Since the device identity is exchanged during the EDHOC session, and the certificate of the device is communicated to the authenticator as part of the Voucher Response message, there is no need to transport the device identity in OSCORE messages.
The authenticator playing the role of the {{RFC9031}} JRC obtains the device identity from the execution of the authorization protocol.

### Flight 4

Flight 4 is the OSCORE response carrying CoJP response message.
The message is processed as specified in {{Section 8.4.2 of RFC9031}}.

# Examples of protocol execution
This section presents high level examples of the protocol execution.

Note: the examples below include samples of access policies used by W. These are provided for the sake of completeness only, since the authorization mechanism used by W is out of scope in this document.

## Minimal {#example_minimal}
This is a simple example that demonstrates a successful execution of ELA.

Premises:

- device u1 has ID_U = key id = 14
- the access policy in W specifies, via a list of ID_U, that device u1 can enroll via any domain authenticator, i.e., the list contains ID_U = 14.
In this case, the policy only specifies a restriction in terms of U, effectively allowing enrollment via any V.

Execution:

1. device u1 discovers a gateway (v1) and tries to enroll
2. gateway v1 identifies the zero-touch join attempt by checking that the label of EAD_1 = -TBD1, and prepares a Voucher Request using the information contained in the value of EAD_1
2. upon receiving the request, W obtains ID_U = 14, authorizes the access, and replies with Voucher Response

## Wrong gateway {#example_wrong_gateway}
In this example, a device u1 tries to enroll a domain via gateway v1, but W denies the request because the pairing (u1, v1) is not configured in its access policies.

This example also illustrates how the REJECT_INFO field of the EDHOC error Access Denied could be used, in this case to suggest that the device should select another gateway for the join procedure.

Premises:

- devices and gateways communicate via Bluetooth Low Energy (BLE), therefore their network identifiers are MAC addresses (EUI-48)
- device u1 has ID_U = key id = 14
- there are 3 gateways in the radio range of u1:
  - v1 with MAC address = A2-A1-88-EE-97-75
  - v2 with MAC address = 28-0F-70-84-51-E4
  - v3 with MAC address = 39-63-C9-D0-5C-62
- the access policy in W specifies, via a mapping of shape (ID_U; MAC1, MAC2, ...) that device u1 can only join via gateway v3, i.e., the mapping is: (14; 39-63-C9-D0-5C-62)
- W is able to map the PoP key of the gateways to their respective MAC addresses

Execution:

1. device u1 tries to join via gateway v1, which forwards the request to W
2. W determines that MAC address A2-A1-88-EE-97-75 is not in the access policy mapping, and replies with an error. The error_content has REJECT_TYPE = 1, and the plaintext OPAQUE_INFO (used to compute the encrypted REJECT_INFO) specifies a list of suggested gateways = \[h'3963C9D05C62'\]. The single element in the list is the 6-byte MAC address of v3, serialized as a bstr.
3. gateway v1 assembles an EDHOC error "Access Denied" with error_content, and sends it to u1
4. device u1 processes the error, decrypts REJECT_INFO, and retries the protocol via gateway v3


# Acknowledgments
{: numbered="no"}

The authors sincerely thank {{{Aurelio Schellenbaum}}} for his contribution in the initial phase of this work, and Marco Tiloca for extensively reviewing the document.
We also thank Christian Amsüss for his active participation in discussions that led to improvements in the document.

Work on this document has in part been supported by the Horizon Europe Framework Programme project OpenSwarm (grant agreement No. 101093046).

--- fluff
