---
dip: 10
title: DiemID Spec
authors: Sunmi Lee (@sunmilee), David Wolinsky (@davidiw), Andrey Chursin(@andll), Kevin Hurley (@kphfb)
status: Draft
type: Informational
created: 11/03/2020
last_updated: 04/28/2021
issue: https://github.com/diem/dip/issues/156
---

import useBaseUrl from '@docusaurus/useBaseUrl';

# Summary

This DIP describes DiemID, a human-readable identifier for user accounts, and a protocol standard for pre-flight exchange using DiemID to form payment transactions.   

# Motivation
DiemID provides a convenient method for identifying users within a VASP. DiemID allows users to exchange human-readable identifiers as either the sender or the receiver of peer-to-peer payments, and plays the role of an email address for payments. The benefits of using a DiemID are:
* Privacy: DiemID's do not appear on-chain. The standard uses a pre-flight off-chain exchange between VASPs to agree on a reference ID. The sending VASP can then submit an on-chain transaction containing only this reference ID and without any potentially identifiable user information.
* Persistent Identifiers: Currently there are no persistent user identifiers in the Diem ecosystem. DiemID establishes a persistent identifier from a user's perspective which are not used publicly on-chain.

# End-to-End Experience
Below is an example of using DiemID for transferring money from one user to another. 

## Prerequisite:
* VASPs get approval from association (via some offline process) on domain name. 
* VASPs have access to a storage solution mapping domain IDs to parent VASP address
```
avasp, 0xf72589b71ff4f8d139674a3f7369c69b
bvasp, 0xc5ab123458df0003415689adbb47326d
```

## An Example P2P Transaction Flow: 
* Bob wants to receive funds from Alice
* Alice has an account with VASP A: `alice@avasp`
* Bob has an account with VASP B: `bob@bvasp`
* Bob shares the DiemID, `bob@bvasp` with Alice
* Alice logs into VASP A, enters Bob’s identifier, an amount to pay, and submits payment
* VASP A contacts VASP B via an off-chain API with the sender identifier `alice@avasp` and requests a reference_id, `rb`
* VASP A constructs a transaction with the specified amount and the reference_id `rb` and submits it to the Diem network
* VASP B receives a transaction with metadata containing `rb`, deposits the amount in Bob's account. (VASP B may use Alice’s DiemID for its own record keeping or seek receipt confirmation from Bob.)


# DiemID Format
Diem defines globally unique identifiers to facilitate transacting on the Diem Payment Network. As Diem stores no personally identifiable information on-chain, these identifiers are exchanged during the off-chain process between two VASPs to identify a distinct source and destinations within their respective VASPs in order to produce a reference_id that can be stored on-chain. The format of identifiers is `[user_identifier]@[vasp_domain_identifier]`

Example: `alice@avasp`

* `user_identifier` is a reusable identifier that represents either the source or destination end-user of a payment transaction. It is unique to per user at VASP level. Specification:
  * Case insensitive
  * Valid regular expression: `^[a-zA-Z0-9][a-zA-Z0-9.]*$`
  * Maximum length: 64 characters
* `vasp_domain_identifier` is a unique string that is mapped to a VASP. Specification:
  * Case insensitive
  * Valid regular expression: `^[a-zA-Z0-9][a-zA-Z0-9.]*$`
  * Maximum length: 63 characters (64 including `@`)

We are starting with a small character set to keep it compatible with most naming schemes, but may add on to the set in the future. 

# On-chain data
## DiemID Domain

```
resource struct DiemIdDomains {
    domains: vector<DiemIdDomain>,
}

struct DiemIdDomain {
    domain: vector<u8>,  // Utf-8 encoded
}
```
* Fields definition:
   * `domain` - name of a DiemID domain 
* The `DiemIdDomains` resource can only be published into an account with `Role == PARENT_VASP_ROLE_ID`.
* The `DiemIdDomains` contains a list of `DiemIdDomain` structs that are associated with a VASP. As such, it is possible to register more than one DiemID Domain for a given VASP
* Only the special TreasuryCompliance account (address `0xB1E55ED`) can manipulate a DiemIdDomains resource:
  * Only TC account can create and publish a `DiemIdDomains` resource
  * Only TC account can add, remove or update a `DiemIdDomain` within `DiemIdDomains` resource
* `DiemIDDomains` resource will be created in an empty state as part of creating a `ParentVASP` account resource, and existing `ParentVASP` accounts without `DiemIDDomains` will have the resource instantiated by the DiemRoot account
* In order to register a DiemID domain, a VASP needs to submit a request to Diem Association. After approval, Diem Association will use TreasuryCompliance account to issue an on-chain transaction to register a DiemID Domain

## DiemID Domain Events

The Move contract that manages DiemID domains must emit an event every time DiemID domain is created, removed or updated. Those events are critical for applications to be able to efficiently index existing DiemID domains.
An application can be built to listen for these events to construct a mapping of DiemID domains to VASP accounts for lookup of onchain addresses given a DiemID domain.
While DiemID domains are published into VASP account resource, DiemID domain events are published under the TreasuryCompliance account. We consolidate events under single account to allow indexers to follow a single event stream.

To store events, `DiemIdDomainManager` resource is published under the TreasuryCompliance account(address `0xB1E55ED`).

```
resource struct DiemIdDomainManager {
    /// Events are emmitted 
    diem_id_domain_events: Event::EventHandle<Self::DiemIdDomainEvent>,
}

struct DiemIdDomainEvent {
    removed: bool,
    domain: DiemIdDomain,
    address: address,
}
```  

* Fields definition:
  * `removed` - `true`, if DiemIdDomain was removed, `false` if it was created or updated
  * `domain` - exact copy of `DiemIdDomain` that was added/removed from `DiemIdDomains` resource of a VASP account
  * `address` - address of an account where `DiemIdDomain` was added or removed


# ReferenceID Exchange
Transactions with PaymentMetadata require a ReferenceID in order to submit a transaction to the chain. PaymentMetadata reveal nothing distinctly about either the sender or receiver and do not create a linkability between the sender or receiver across payments. Hence, the ReferenceID must be established off-chain and this protocol defines one such way to do so leveraging DiemID.

A VASP intending to send a payment to another using DiemIDs constructs a ReferenceIDCommand and transmits it to the receiver. If the receiver knows the recipient DiemID and can potentially accept payments, it returns a success. Otherwise, it returns an error indicating the reason.

In the case that the amount to be sent would exceed the limit of the travel rule, the sending party should follow this exchange with a PaymentCommand using the same reference_id and specify the sending and receiving subaddresses as all 0.

The format of the command is:

```
{
   "_ObjectType": "CommandRequestObject",
    "command_type": "ReferenceIDCommand",
    "command": {
	    "_ObjectType": "ReferenceIDCommand",
	    "sender": "alice@avasp",
	    "sender_address": "dm1pptdxvfjck4jyw3rkfnm2mnd2t5qqqqqqqqqqqqq305frg",
	    "receiver": "bob@bvasp",
	    "reference_id": "5b8403c9-86f5-3fe0-7230-1fe950d030cb", 
    },
    "cid": "12ce83f6-6d18-0d6e-08b6-c00fdbbf085a",
}
```

**CommandRequestObject:**

| Field 	    | Type 	     | Required? 	| Description 	           |
|-------	    |------	     |-----------	|-------------	           |
| _ObjectType   | str        | Y | Fixed value: `CommandRequestObject`|
| command_type  | str        | Y | Fixed value: `ReferenceIDCommand`|
| command       | Command object | Y | The Command to request. In this DIP, refers to ReferenceIDCommandObject |
| cid           | str         | Y            | A unique identifier for the Command. Should be a UUID according to [RFC4122](https://tools.ietf.org/html/rfc4122) with "-"'s included. |

**ReferenceIDCommandObject:**

| Field 	    | Type 	     | Required? 	| Description 	           |
|-------	    |------	     |-----------	|-------------	           |
| _ObjectType   | str    | Y | Fixed value: `ReferenceIDCommand`|
| sender        | str          | Y            | Sender's full DiemID |
| sender_address| str          | Y            | Sender's onchain [account identifier](https://github.com/diem/dip/blob/main/dips/dip-5.md) with subaddress set to `None` or the zero subaddress|
| receiver     | str          | Y            | Receiver's full DiemID |
| reference_id  | str          | Y            | Reference ID of this transaction to be included into payment metadata |


The format of the success response is:
```
{
   "_ObjectType": "CommandResponseObject",
    "status": "success",
    "result": {
	    "_ObjectType": "ReferenceIDCommandResponse",
	    "receiver_address": "dm1p7ujcndcl7nudzwt8fglhx6wxn08kgs5tm6mz4us2vfufk",
    },
    "cid": "12ce83f6-6d18-0d6e-08b6-c00fdbbf085a",
}
```

**ReferenceIDCommandResultObject:**

| Field 	    | Type 	     | Required? 	| Description 	           |
|-------	    |------	     |-----------	|-------------	           |
| _ObjectType   | str        | Y | Fixed value: `ReferenceIDCommandResponse`|
| receiver_address       | str | Y | Receiver's onchain [account identifier](https://github.com/diem/dip/blob/main/dips/dip-5.md) with subaddress set to `None` or the zero subaddress |

#### Error Codes

`duplicate_reference_id`: Duplicate Reference ID was rejected by the receiving end

`invalid_receiver`: Receiving end could not find the user with the given user_identifier

`invalid_field_value`: Reference ID is in the wrong format. See more details in [DIP-1](https://github.com/diem/dip/blob/main/dips/dip-1.mdx#request-object-validation-error-codes)

## On-Chain Transaction Settlement
If the amount is below the travel rule limit, the sending VASP can send a p2p transaction with PaymentMetadata and the `reference_id` VASPs agreed on to settle the transaction. 
```
enum PaymentMetadata {
    PaymentMetadataV0(ReferenceId),
}
type ReferenceId = [u8, 16];
```

If the amount is at or above the travel rule limit, VASPs should trigger an off-chain protocol for KYC exchange. The same `reference_id` must be used to perform a travel rule exchange as described in [DIP-1](https://github.com/diem/dip/blob/master/dips/dip-1.md).
