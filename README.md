---
eip: <to be assigned>
title: Non-Tradable Tokens for wallet KYC
description: A standard interface for Soulbound Token using in wallet KYC.
author: Louis Chan <blocklab123@gmail.com>
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2022-07-01
---

# EIP-?: Non-Tradable Tokens for wallet KYC

## Abstract
Blue Check is a unique, non-tradable, non-transferable token representing a wallet owned by someone who has gone through a standard KYC process. This KYC token standard protects owners' privacy and fulfills regulation requirements. This proposal also includes a revoke mechanism in case the verified wallet is compromised. All Blue Check tokens are unique; one person can only generate one token using their personal information. Those tokens do not have a monetary value and only serve as proof that this wallet passed a specific KYC process.


## Motivation
In some countries, KYC is required for Defi, NFT trading, and exchange withdrawal. By providing a standard interface for non-tradable KYC tokens, we allow more applications to offer their services without violating laws in certain countries. The KYC tokens also need to be able to protect privacy while revealing the minimum required personal information of the account owners.



## Specification
The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

The KYC system comprises three main parts:
- The Blue Check (Soulbound Token for KYC)
- The OKEY (A transferrable token for revoking the Blue Check)
- The Control Hub


The Blue Check is a unique, non-tradable, non-transferable token bound with an address. The Blue Check contains the following information
- hash of first name, last name, date of birth, id number (personal hash)
- nationality

Each Blue Check will pair with an OKEY; this is a transferrable token. OKEY is minted together with Blue Check. Users should store their OKEY in a safe address immediately after receiving it. Burning the OKEY will revoke the Blue Check status.

The Control Hub is a smart contract that controls both the minting and revoking Blue Check. 

Minting a Blue Check is a three-step process:
- Users need to connect their wallets to an authorized KYC solution provider.
- Users submit their KYC applications to the solution provider with an anti-fraud process.
- The solution provider confirms the document's authenticity and generates the personal hash to mint both Blue Check and OKEY for users.

### Personal hash algorithm
keccak256(first_name+last_name+DOB+id)

### The Authorizer specification
The Authorizer contract exposes the following functions:
```solidity
// SPDX-License-Identifier: MIT
 
pragma solidity ^0.8;
 
 
interface IAuthorizer /* is ERC165 */ {
 
   /// Event emitted when a pair of tokens with same `tokenId` are minted to `owner`
   event MintedKeyPair(address _owner, uint256 _tokenId, string _identityHash);
 
   /// Event emitted when a pair of tokens with same `tokenId` of `owner` are revoked by `owner`
   event Revoked(address _owner, uint256 tokenId, string _identityHash);
 
   /// Event emitted when a pair of tokens with same `tokenId` of `owner` are revoked by `authorizer`
   event HardRevoked(address _authorizer, address _owner, uint256 tokenId, string _identityHash);
 
   /// @notice Revoke a pair of tokens by Okey token owner
   /// @dev Throws if `_tokenId` not exist. Throws if `msg.sender` is not the Okey owner.
   /// Emits the Revoked event.
   /// @param _tokenId Identifier of the token
   function revoke(uint _tokenId) external;
 
   /// @notice Revoke a pair of tokens by the authorizer
   /// @dev Throws if `_identityHash` not exist.
   /// Throws if blueCheck token is not exist.
   /// Throws if `msg.sender` is not the Authorizer contract owner.
   /// Emits the Revoked event.
   /// @param _identityHash the identity hash of the token owner
   function hardRevoke(string memory _identityHash) external;
}

```

### The OKEY specification
Same as ERC721

### The Blue Check specification
The Blue Check contract MUST include the following interfaces.

```solidity
// SPDX-License-Identifier: MIT
 
pragma solidity ^0.8;
 
 
interface IBlueCheck /* is ERC165 */ {
 
 
   /// Event emitted when token with `tokenId` is transfered from the `_from` address
   /// to the `_to` address operated by the `authorizer` address
   event DelegatedTransfer(address _authorizer, address _from, address _to, uint _tokenId);
 
   /// @notice delegate an authorizer to manage the minting and revoke action for this token
   /// @dev Throws if `msg.sender` is not the contract owner.
   /// @param _authorizer the address of an authorizer
   function delegateAuthorizer(address _authorizer) external;
 
   /// @notice Count all tokens assigned to an owner
   /// @param _owner query the balance of the `_owner` address
   /// @return Number of tokens owned by `owner`
   function balanceOf(address _owner) external view returns (uint256);
 
   /// @notice Get owner address of a token
   /// @param _tokenId Identifier of the token
   /// @return Address of the owner of `_tokenId`
   function ownerOf(uint256 _tokenId) external view returns (address);
  
   /// @notice Transfer ownership of a token -- THE CALLER IS RESPONSIBLE
   ///  TO CONFIRM THAT `_to` IS CAPABLE OF RECEIVING NFTS OR ELSE
   ///  THEY MAY BE PERMANENTLY LOST
   /// @dev Throws unless `msg.sender` is the authorized operator for this token.
   /// Throws if `_from` is not the current owner.
   /// Throws if `_tokenId` is not a valid NFT.
   /// Emits the DelegatedTransfer event.
   /// @param _from The current owner of the token
   /// @param _to The new owner
   /// @param _tokenId The token to transfer
   function delegatedTransferFrom(address _from, address _to, uint _tokenId) external;
 }
```


## Rationale
As EIP-721 tokens have seen widespread adoption with wallet providers and marketplaces, using its ERC721Metadata interface with EIP-165 for feature-detection potentially allows implementers to support Blue Check tokens out of the box.

### Revoke Mechanism
If a user's private to an account or a contract gets compromised or rotated, a user may lose the ability to associate themselves with the Blue Check. Therefore, EIP-X implementers should build re-issuance and revocation processes to enable recourse. 

The ideal case will be implementing strictly decentralized, permissionless, and censorship-resistant re-issuance processes. However, we recommend providing the following revoke mechanism without such a DAO.

### Revoke by the OKEY
OKEY is supposed to store in a separate wallet address. If the address bound with Blue Check is compromised, users can use the OKEY to revoke their Blue Check, and the revocation will burn both OKEY and Blue Check.

### Revoke by the issuer
We need to consider the case that users lose both their Blue Check wallet and OKEY wallet. Then users will be required to re-do the verification process and trigger the hard revoke by the issuer. After confirming the document's authenticity and generating the personal hash to revoke the Blue Check using "function X"


## Backwards Compatibility
We have intentionally adopted the [`EIP-165`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-165.md)
 and ERC721Metadata functions to create a high degree of backward compatibility with [`EIP-721`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md). We have deliberately used [`EIP-721`](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md)
terminology such as function ownerOf(...) or function balanceOf(...) to minimize the friction of integrating 


## Reference Implementation

You can find an implementation of this standard in [/assets](/assets).

## Copyright

Copyright and related rights waived via [CC0](https://github.com/ethereum/EIPs/blob/master/LICENSE.md).
