---
title: Lightweight Authorization for Authenticated Key Exchange.
abbrev: Lightweight Authorization for AKE.
docname: draft-selander-ace-ake-authz-latest

ipr: trust200902
cat: info

coding: utf-8
pi: # can use array (if all yes) or hash here
  toc: yes
  sortrefs: yes
  symrefs: yes
  tocdepth: 2

author:
      -
        ins: G. Selander
        name: Göran Selander
        org: Ericsson AB
        email: goran.selander@ericsson.com
      -
        ins: J. Mattsson
        name: John Mattsson
        org: Ericsson AB
        email: john.mattsson@ericsson.com
      -
        ins: M. Vucinic
        name: Malisa Vucinic
        org: INRIA
        email: malisa.vucinic@inria.fr
      -
        ins: M. Richardson
        name: Michael Richardson
        org: Sandelman Software Works
        email: mcr+ietf@sandelman.ca




normative:


informative:

  RFC3748:
  RFC7228:
  RFC8613:
  I-D.ietf-lake-reqs:
  I-D.raza-ace-cbor-certificates:
  I-D.irtf-cfrg-hpke:
  


  HKDF:
    target: https://eprint.iacr.org/2010/264.pdf
    title: "Cryptographic Extraction and Key Derivation: The HKDF Scheme"
    author:
      -
        ins: H. Krawczyk
    date: May 2010


      
--- abstract

This document describes a procedure for augmenting an authenticated Diffie-Hellman key exchange with third party assisted authorization targeting constrained IoT deployments (RFC 7228).

--- middle

# Introduction  {#intro}


For constrained IoT deployments {{RFC7228}} the overhead contributed by security protocols may be significant which motivates the specification of lightweight protocols that are optimizing, in particular, message overhead (see {{I-D.ietf-lake-reqs}}). This document describes a lightweight procedure for augmenting an authenticated Diffie-Hellman key exchange with third party assisted authorization. 

The procedure involves a device, a domain authenticator and a AAA server. The device performs mutual authentication and authorization of the authenticator, assisted by the AAA server which provides relevant authorization information in the form of a "voucher".

The protocol specified in this document optimizes the message count by performing authorization and enrolment in parallel with authentication, instead of in sequence which is common for network access control. It further reuses protocol elements from the authentication protocol leading to reduced message sizes on constrained links. 

The specification assumes a lightweight AKE protocol {{I-D.ietf-lake-reqs}} between device and authenticator, and defines the integration of a lightweight authorization procedure. This enables a secure target interaction in few message exchanges. In this document we consider the target interaction to be "enrolment", for example certificate enrolment or joining a network for the first time, but it can be applied to authorize other target interactions.

This protocol is applicable in a wide variety of settings, e.g. an enterprise network using EAP {{RFC3748}}. 

# Problem Description {#prob-desc}

The (potentially constrained) device wants to enrol into a domain over a constrained link. The device authenticates and enforces authorization of the (non-constrained) domain authenticator with the help of a voucher, and makes the enrolment request. The domain authenticator authenticates the device and authorizes its enrolment. Authentication between device and domain authenticator is made with a lightweight authenticated Diffie-Hellman key exchange protocol (LAKE, {{I-D.ietf-lake-reqs}}). The procedure is assisted by a (non-constrained) AAA server located in a non-constrained network behind the domain authenticator providing information to the device and to the domain authenticator. 

The objective of this document is to specify such a protocol which is lightweight over the constrained link and reuses elements of the LAKE. See illustration in {{fig-overview}}.


~~~~~~~~~~~
                   Voucher
              LAKE  Info                
+------------+  |    |   +---------------+  Voucher  +------------+
|            |  |    |   |               |  Request  |            |
|            |--|----o-->|    Domain     |---------->|    AAA     |
|   Device   |<-|---o----| Authenticator |<----------|   Server   |
|            |--|---|--->|               |  Voucher  |            |
|            |      |    |               |  Response |            |
+------------+      |    +---------------+           +------------+
                  Voucher


~~~~~~~~~~~
{: #fig-overview title="Overview and example of message content. The voucher info and voucher are sent together with LAKE messages." artwork-align="center"}


# Assumptions

## Device

The device is pre-provisioned with an identity ID and asymmetric key credentials: a private key, a public key (PK_D), and optionally the public key certificate issued by the manufacturer Cert(PK_D), used to authenticate to the domain authenticator. The ID may be a reference or pointer to the certificate. 

The device is also provisioned with information about its AAA server:

* At least one static public DH key of the AAA server (G_S).
* Location information about the AAA server (Loc_S), e.g. its domain name. This information may be available in the device's manufacturer certificate Cert(PK_D).



## Domain Authenticator

The domain authenticator has a private key and corresponding public key PK_A used to authenticate to the device. 

The domain authenticator needs to be able to locate the AAA server of the device for which the Loc_S is expected to be sufficient. The communication between domain authenticator and AAA server is mutually authenticated and protected. Authentication credentials used with the AAA server is out of scope. How this communication is established and secured (typically TLS) is out of scope. 


## AAA Server

The AAA server has a private DH key corresponding to G_S to decrypt the identity of the device (see {{p-as}}). Authentication credentials used with the domain authenticator is out of scope.

The AAA server provides the authorization decision for enrolment to the device in the form of a (potentially signed) CBOR encoded voucher. The AAA server provides information to the domain authenticator about the device, such as the the device's certificate Cert(PK_D).

The AAA server need to be available during the execution of the protocol. 


## Lightweight AKE

We assume a Diffie-Hellman key exchange protocol complying with the LAKE requirements {{I-D.ietf-lake-reqs}}. Specifically we assume for the LAKE:

* Three messages
* CBOR encoding
* The ephemeral public Diffie-Hellman key of the device, G_X, is sent in message 1
* The static public key of the domain authenticator, PK_A, sent in message 2
* Support for Auxilliary Data AD1-3 in messages 1-3 as specified in section 2.5 of {{I-D.ietf-lake-reqs}}.
* Cipher suite negotiation where the device can propose ECDH curves restricted by its available public keys of the AAA server.



# The Protocol

Three security sessions are going on in parallel (see figure {{fig-protocol}}):

* Between device and (domain) authenticator,
* between authenticator and AAA server, and 
* between device and AAA server mediated by the authenticator. 

We study each in turn, starting with the last.

~~~~~~~~~~~
 Device                        Authenticator           AAA Server
   |                                 |                     |            
   |--- Message 1 incl. G_X, AD1 --->|--- Voucher req. --->|
   |                                 |                     |
   |<-- Message 2 incl. PK_A, AD2 --|<-- Voucher resp. ---|          
   |                                 |                     
   |--- Message 3 incl. AD3 -------->|                    

~~~~~~~~~~~
{: #fig-protocol title="The Protocol" artwork-align="center"}


In the following, please note that G_X has multiple roles:

   * ephemeral key of the LAKE protocol between device and authenticator, and
   * ephemeral key and nonce in the ECIES scheme between device and AAA server.


## Device <-> AAA Server {#p-as}

The communication between device and AAA server is carried out via the authenticator protected using an ECIES hybrid encryption scheme (see {{I-D.irtf-cfrg-hpke}}): The device uses the private key of its ephemeral DH key G_X generated for LAKE message 1 (see {{p-r}}) together with the static public DH key of the AAA server G_S to generate a shared secret G_XS. The shared secret is used to derive AEAD encryption keys to protect data between device and AAA server exchanged in AD1 and AD2 (between device and authenticator) and in voucher request/response (between authenticator and AAA server).

TODO: Reference relevant ECIES scheme in {{I-D.irtf-cfrg-hpke}}.

TODO: Define encryption keys for both directions

AD1 contains:

* Location information about the AAA server, Loc_S
* Crypto context, CC, for the communication between device and AAA server
* The encrypted identity of the device AEAD(ID), with CC as Additional Data. 

AD1 SHALL be a CBOR sequence as defined below:

~~~~~~~~~~~
AD1 = (
    LOC_S:           tstr,
    CC:              int,
    CIPHERTEXT_ID:   bstr,
)
~~~~~~~~~~~

TODO Describe how is CIPHERTEXT_ID generated

AD2 contains the voucher. 


### Voucher

The voucher is the output from the AEAD with empty plaintext and the following Additional Data:

* Type - indicating kind of voucher.
* PK_A 
* G_X 
* CC 
* ID

All parameters except 'Type' are as received from in the voucher request (see {{r-as}}). As it is the output of the AEAD, the voucher is integrity protected between AAA server and device.

The AEAD Additional Data SHALL be a CBOR array as defined below:

~~~~~~~~~~~
Voucher_Additional_Data = [
    voucher_type:  int,
    PK_A:         bstr,
    G_X:           bstr,
    CC:            int,
    ID:            bstr,
]
~~~~~~~~~~~

TODO: CBOR encoding of the voucher. Consider making the voucher a CBOR Map to indicate type of voucher, to indicate the feature (cf. {{r-as}})



## Device <-> Authenticator {#p-r}

The device and authenticator run the LAKE protocol authenticated with public keys (PK_D and PK_A) of the device and the authenticator. The normal processing of the LAKE is omitted here.


### Message 1

#### Device processing

The device selects a cipher suite with an ECDH curve satisfying the static public DH key G_S of the AAA server. As part of the normal LAKE processing, the device generates the ephemeral public key G_X to be sent in LAKE message 1. This is the same ephemeral key as used in the ECIES scheme in {{p-as}}. A new G_X MUST be generated for each execution of the protocol. 

The device sends LAKE message 1 with AD1 as specified in {{p-as}}.


#### Authenticator processing

The authenticator receives LAKE message 1 from the device, which triggers the exchange of voucher related data with the AAA server as described in {{r-as}}.


### Message 2

#### Authenticator processing

The authenticator sends LAKE message 2 to the device with the voucher (see {{p-as}}) in AD2. The public key PK_A is included in the way public keys are included in the LAKE protocol.


TODO: Encoding of PK_A

 
#### Device processing
 
The device MUST verify the voucher using its ephemeral key G_X sent in message 1 and PK_A received in message 2. If the voucher does not verify, the device MUST discontinue the protocol.


### Message 3

#### Device processing


The device sends message 3. AD3 depends on the kind of enrolment the device is requesting. It may e.g. be a CBOR encoded Certificate Signing Request, see {{I-D.raza-ace-cbor-certificates}}.

#### Authenticator processing

The authenticator receives message 3.


## Authenticator <-> AAA Server {#r-as}

The authenticator and AAA server are assumed to have secure communication, for example based on TLS authenticated with certificates. 


### Voucher Request


The authenticator sends the voucher request to the AAA server with the following content:

* PK_A 
* G_X 
* CC 
* AEAD(ID)

The voucher request SHALL be a CBOR array as defined below:

~~~~~~~~~~~
Voucher_Request = [
    PK_A:           bstr,
    G_X:             bstr,
    CC:              int,
    CIPHERTEXT_ID:   bstr,
]
~~~~~~~~~~~

### Voucher Response

The AAA server decrypts the identity of the device and looks up its certificate, Cert(PK_D). The AAA server send the voucher response to the AAA server containing

* Cert(PK_D)
* Voucher (see {{p-as}})

The voucher response SHALL be a CBOR array as defined below:

~~~~~~~~~~~
Voucher_Response = [
    CERT_PK_D:     bstr,
    Voucher:        bstr,
]
~~~~~~~~~~~

TODO: The voucher response may contain a "Voucher-info" field as an alternative to make the voucher a CBOR Map (see {{p-as}})

# Security Considerations  {#sec-cons}

TODO: Identity protection of device

TODO: How can the AAA server attest the received PK_A?

TODO: Remote attestation 

# IANA Considerations  {#iana}

TBD CC registry

TBD voucher type registry

--- back


