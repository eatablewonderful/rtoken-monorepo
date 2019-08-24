RToken Ethereum Contracts
=========================

`RToken`, or Reedemable Token, is an _ERC20_ token that is 1:1 redeemable to its
underlying _ERC20_ token. The underlying tokens are invested into interest
earning assets specified by the allocation strategy, for example into
[_Compound_](http://compound.finance). Owners of the _rTokens_ can use a
definition called _hat_ to configure who is the beneficiary of the accumulated
interest. _RToken_ can be used for community funds, charities, crowdfunding,
etc. It is also a building block for _DApps_ that need to lock underlying tokens
while not losing their earning potentials.

## How does it look like

```
            +------+
      +-----+ User +-------+
      |     +------+       |
      |                    |
      |                    |                      +-----------------------+
      |                    |                      |compound.finance cToken|
 +----+-----+  +-----------+-----------+          +-----------------------+
 |  Dapp    |  |ERC20 Compatible Wallet|                     ^
 +----+-----+  +-----------+-----------+                     |
      |                    |                     +--------------------------+
      |                    |                     |CompoundAllocationStrategy|
      |                    |                     +--------------------------+
      |                    |                                 ^
      |                    v                                 |
      |    +----------------------------------+              |
      |    |          RToken                  |              |
      |    |----------------------------------|              |
      |    | - ERC20 compatbile               |              |
      +--->| - Mint/Redeem/PayInterst         |              |
           | - "Hat"/Beneficiary system       |              |
           | - Changeable allocation strategy +--------------+
           | - Configurable parameters        | IAllocationStrategy interface
           | - Admin role (human/DAO)         |
           +----------------------------------+
                       ^
                       |
                    +--+--+
                    |Admin|
                    +-----+
```

## What does it do

As an example, let's pick [_DAI_](https://dai.makerdao.com/) as our underlying
token contract. As a result, the _rToken_ instantiation is conveniently called
_rDAI_ in this example.

### 1. Hat Types

A hat defines who can keep the interest generated by the underlying _DAI_
deposited by users.
Every address can be configured with one and only one hat, but a hat can have
multiple beneficiaries.

There are three kinds of hats:

* `Zero Hat` - It is the default hat for all addresses (even before they have a
balance).
Any interest generated by the _DAI_ tokens locked by the owner are entitled to
the owner himself. This kind of hat is subjected to the _Hat Inheritance Rules_

* `Self Hat` - Similar to the _Zero Hat_, as the owner keeps all _DAI_ tokens
and generated interest. However in this case it is a deliberate choice by the
owner, hence the _Hat Inheritance Rules_ do not apply to this address.

* `Other Hat` - This hat can be inherited or created by the user. The interest
generated by the _Hat_ can be withdrawn to the address of any recipient
indicated in the hat definition. _Hat Inheritance Rules_ do not apply to this
address.

### 2. Hat Definition

A _hat_ is defined by a list of recipients, and their relative proportions for
splitting the _rDAI_ loans from the owner.

For example:
```
{
    recipients: [A, B],
    proportions: [90, 10]
}
```
defines that the _DAI_ tokens will be loaned to address A and address B in the
relative proportions of 90:10, effectively A receives 90% and B receives 10% of
the generated interest.

### 3. Mint

The user first needs to approve the _rDAI_ contract to use its _DAI_ tokens,
then the user can mint as much _rDAI_ as they have _DAI_. One _rDAI_ is always
equal to one _DAI_.

As a result, the _DAI_ tokens transferred in order to mint new _rDAI_ tokens are
invested automatically into the _Saving Strategy_, and the recipients indicated
in the user's chosen hat can withdraw any generated interest.

### 4. Redeem

Users may redeem the _DAI_ tokens they deposited at any time by transferring
back the _rDAI_ tokens.

As a result, the invested _DAI_ tokens are recollected from the recipients, and
given back to the owner.

### 5. Transfer

_rDAI_ contract is _ERC20_ compliant, and one should use _ERC20_ _transfer_ or
_approve_ functions to transfer the _rDAI_ tokens between addresses.

As a result, the amount of _DAI_ tokens loaned out by the source relevant to the
transaction is recollected, and loaned to the new recipients according to the
hat of the destination.

### 6. Pay Interest

Recipients of loaned _DAI_ tokens are entitled to the full amount of interest
earned from them.

Anyone can call the _payInterest_ function, which converts the earned interest
to new _rDAI_ tokens for the recipient. This mechanism allows contract addresses
to also be recipients, despite not having implemented functions to call the
_payInterest_ function externally.

Unlike the mint processes, _rDAI_ generated in this process does not loan equal
amount of _DAI_ tokens to any recipient. The owner may choose to loan them by
using _loanInterest_, or transfer the _rDAI_ to another address and trigger the
hat switching process.

Interest payment rules may apply as per configuration (see _Governance
section_).

### 6. Hat Inheritance Rules

In order to maximize the cause the hat owners choose, the following rules are
stipulated in order to allow hats to spread to new users:

* All addresses have the _Zero Hat_ by default.

* During the mint process, a user's deposited _DAI_ tokens are loaned to the
recipients as indicated in the user's hat. If the recipient has the _Zero Hat_,
the recipient will inherit the minter's hat.

* During the transfer process, _DAI_ tokens are recollected and loaned to the
new recipients. If the recipient has the _Zero Hat_, the recipient will inherit
the source's hat.

For example: Alice sets UNICEF France as recipients of her generated interest.
Bob has never used _rDAI_, and thus has a _Zero Hat_. When Alice sends Bob 100
_rDAI_, Bob inherits Alice's hat, and UNICEF France keep accruing interest. Bob
then sends the 100 _rDAI_ along to Charlie. But Charlie already has a hat, so
the underlying 100 _DAI_ are now loaned to Charlie's chosen recipients.

### 7. Hats for contract addresses

As most contract addresses can't execute arbitrary functions, they can generally
only change hat once, from inheritance by the first user to send _rDAI_ to the
contract address.
Because it is sometimes unclear who the owner of a contract is, the `rToken`
contract allows the admin to change the hat of any contract address.

##### Note that the addresses without code are assumed to be able to demonstrate
the ownership by indisputable ownership of the private key, so even _admin_ is
not allowed to change that for them.

In order to avoid needing to use the admin, we advise buidlers who are looking
to accept _rDAI_ to set up their contracts correctly by:
1. getting some _rDAI_ for themselves
2. selecting or creating a hat of their choosing
3. transferring any amount of _rDAI_ to their contracts

### 8. Allocation Strategy

The _IAllocationStrategy_ interface defines what _RToken_ can integrate for
investing the underlying assets in exchange for saving assets that earns
interest.

It is changeable by admin. Per request, the _rDai_ contract will redeem all
underlying assets at once from old allocation strategy, and invest all into new
allocation strategy.

_CompoundAllocationStrategy_ is one implementation. In case of _rDai_, it is
_cDai_.

While it is not possible to forbid admin from using risky strategy, and risk
strategy could cause redeemability to fail if the strategy has heavy losses,
it is up to the admin to make a sensible choice of what consists of a
proper allocation strategy.

### 9. Admin & Governance

The `RToken` contract has an admin role who can:

- Change allocation strategy
- Configure interest payment rules:
  - Minimal threshold of "interest amount / loaned tokens"
  - Minimal period before first interest payment
  - Minimal gap between interest payments
- Change hat for any contract address

It is up to the `rToken` instantiator to decide the degree of decentralization
of this admin. For maximum decentralization, the admin could be a DAO that is
implemented by a DAO framework such as [Aragon](https://aragon.org/), and the
hat change could be controlled by a arbitration process such as
(Kleros)(https://kleros.io/).

# How is it implemented

TODO...

# Deployed Contracts

## Rinkeby
rDAI (latest): `0x46b7cc7dd18eC78d639f329f3Df0d8AE094cA8b6`

## Kovan
rDAI (latest): `0x82679b504e783270A861711f72d5809feA6fC607`

## Mainnet

rDai (ethberlin): TBD
