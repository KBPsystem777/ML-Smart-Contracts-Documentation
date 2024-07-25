---
description: ManageLife.sol
---

# ML Member's NFT Smart Contract Documentation

## Overview

The ManageLife Member's NFT Smart Contract is designed to manage memberships or home ownerships represented as NFTs (Non-Fungible Tokens) using the ERC-721 standard. This contract integrates with the ManageLife platform to provide functionalities such as issuing and managing NFTs, tracking payment statuses, and enabling staking rewards. The contract also supports interactions with other smart contracts like Life (for token issuance) and Marketplace (for trading).

## Key Features

* **NFT Minting and Burning**: Create and destroy NFTs representing properties or memberships.
* **Payment Tracking**: Track the payment status of properties.
* **Staking Rewards**: Issue staking rewards for fully paid properties.
* **Marketplace Integration**: Enable interactions with a marketplace for buying and selling properties.

## Function Details

#### `constructor()`

* **Purpose**: Initializes the contract with a name and symbol for the NFT.
* **Rules**: Sets the name "ManageLife Member" and the symbol "MLRE" for the NFT collection.

#### `setBaseURI(string memory newBaseUri)`

* **Purpose**: Updates the base URI for all NFTs.
* **Rules**: Can only be called by the owner. Emits `BaseURIUpdated` event with the new URI.

#### `setMarketplace(address payable marketplace_)`

* **Purpose**: Sets the address of the Marketplace contract.
* **Rules**: Can only be called by the owner.

#### `setLifeToken(address lifeToken_)`

* **Purpose**: Sets the address of the Life token contract.
* **Rules**: Can only be called by the owner.

#### `markFullyPaid(uint256 tokenId)`

* **Purpose**: Marks a property as fully paid and initiates staking rewards.
* **Rules**: Can only be called by the owner. Emits `FullyPaid` event. If the property is not owned by the admin, it initializes staking rewards.

## ManageLife Smart Contract Documentation

### Overview

The ManageLife Smart Contract is designed to manage memberships or home ownerships represented as NFTs (Non-Fungible Tokens) using the ERC-721 standard. This contract integrates with the ManageLife platform to provide functionalities such as issuing and managing NFTs, tracking payment statuses, and enabling staking rewards. The contract also supports interactions with other smart contracts like Life (for token issuance) and Marketplace (for trading).

#### Key Features

* **NFT Minting and Burning**: Create and destroy NFTs representing properties or memberships.
* **Payment Tracking**: Track the payment status of properties.
* **Staking Rewards**: Issue staking rewards for fully paid properties.
* **Marketplace Integration**: Enable interactions with a marketplace for buying and selling properties.

### Contract Details

#### SPDX License and Pragma

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;
```

This declares the contract's license and specifies the Solidity version used.

#### Imports

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Burnable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./Life.sol";
import "./Marketplace.sol";
```

The contract imports several OpenZeppelin libraries and two custom contracts (`Life.sol` and `Marketplace.sol`).

#### Contract Definition

```solidity
contract ManageLife is ERC721, ERC721URIStorage, ERC721Burnable, Ownable {
    Life public lifeToken;
    Marketplace public marketplace;

    mapping(uint256 => uint256) public lifeTokenIssuanceRate;
    mapping(uint256 => bool) public fullyPaid;

    event FullyPaid(uint256 tokenId);
    event StakingInitialized(uint256 tokenId);
    event TokenIssuanceRateUpdated(uint256 token, uint256 newLifeTokenIssuanceRate);
    event BaseURIUpdated(string _newURIAddress);
```

This defines the main `ManageLife` contract, which extends several OpenZeppelin contracts for ERC-721 functionality, URI storage, burning, and ownership management.

#### Constructor

```solidity
constructor() ERC721("ManageLife Member", "MLRE") {}
```

Initializes the contract with the name "ManageLife Member" and the symbol "MLRE".

#### Base URI

```solidity
string public baseUri = "https://iweb3api.managelifeapi.co/api/v1/nfts/";

function _baseURI() internal view virtual override returns (string memory) {
    return baseUri;
}

function setBaseURI(string memory newBaseUri) external onlyOwner {
    baseUri = newBaseUri;
    emit BaseURIUpdated(newBaseUri);
}
```

Manages the base URI for the NFTs, allowing the owner to update it.

#### Setting Contract Addresses

```solidity
function setMarketplace(address payable marketplace_) external onlyOwner {
    marketplace = Marketplace(marketplace_);
}

function setLifeToken(address lifeToken_) external onlyOwner {
    lifeToken = Life(lifeToken_);
}
```

Allows the owner to set the addresses of the Marketplace and Life token contracts.

#### Marking Fully Paid Properties

```solidity
function markFullyPaid(uint256 tokenId) external onlyOwner {
    fullyPaid[tokenId] = true;

    if (owner() != ownerOf(tokenId)) {
        lifeToken.initStakingRewards(tokenId);
    }
    emit FullyPaid(tokenId);
}
```

Marks a property as fully paid and initializes staking rewards if the property is not owned by the admin.

#### Minting NFTs

```solidity
function mint(uint256 propertyId, uint256 lifeTokenIssuanceRate_) external onlyOwner {
    require(address(lifeToken) != address(0), "Life token is not set");
    uint256 tokenId = propertyId;
    require(!_exists(tokenId), "Error: TokenId already minted");
    _mint(owner(), propertyId);
    lifeTokenIssuanceRate[tokenId] = lifeTokenIssuanceRate_;
}
```

Mints a new NFT representing a property with a specified issuance rate.

#### Burning NFTs

```solidity
function burn(uint256 tokenId) public override onlyOwner {
    _burn(tokenId);
}
```

Allows the owner to burn an NFT, typically when removing a property from the platform.

#### Retracting and Returning Properties

```solidity
function retract(uint256 tokenId) external onlyOwner {
    _safeTransfer(ownerOf(tokenId), owner(), tokenId, "");
}

function returnProperty(uint256 tokenId) external {
    require(msg.sender == ownerOf(tokenId), "Caller is not the owner");
    safeTransferFrom(msg.sender, owner(), tokenId, "");
}
```

Handles the retraction and return of properties between owners and the admin.

#### Approving Third-Party Access

```solidity
function approve(address to, uint256 tokenId) public override {
    require(
        fullyPaid[tokenId] || ownerOf(tokenId) == owner() || to == address(marketplace),
        "Approval restricted"
    );
    super.approve(to, tokenId);
}
```

Allows homeowners to approve third-party access to their properties under specific conditions.

#### Transfer Hooks

```solidity
function _beforeTokenTransfer(address from, address to, uint256 tokenId) internal override {
    require(
        fullyPaid[tokenId] || from == owner() || to == owner() || msg.sender == address(marketplace),
        "Transfers restricted"
    );
    if (!fullyPaid[tokenId]) {
        if (from == owner()) {
            lifeToken.initStakingRewards(tokenId);
        }
        if (to == owner() && from != address(0)) {
            lifeToken.claimStakingRewards(tokenId);
        }
    }
    emit StakingInitialized(tokenId);

    super._beforeTokenTransfer(from, to, tokenId);
}
```

Enforces rules before transferring NFTs, such as checking payment status and handling staking rewards.

#### Burning and Querying Token URI

```solidity
function _burn(uint256 tokenId) internal override(ERC721, ERC721URIStorage) {
    super._burn(tokenId);
}

function tokenURI(uint256 tokenId) public view override(ERC721, ERC721URIStorage) returns (string memory) {
    return super.tokenURI(tokenId);
}
```

Overrides functions for burning tokens and querying their URIs.

#### Updating Token Issuance Rate

```solidity
function updateLifeTokenIssuanceRate(uint256 tokenId, uint256 newLifeTokenIssuanceRate) external onlyOwner {
    lifeToken.claimStakingRewards(tokenId);
    lifeTokenIssuanceRate[tokenId] = newLifeTokenIssuanceRate;
    lifeToken.updateStartOfStaking(tokenId, uint64(block.timestamp));

    emit TokenIssuanceRateUpdated(tokenId, newLifeTokenIssuanceRate);
}
```

Allows the owner to update the token issuance rate for a property.

### Examples

#### Minting an NFT

Suppose the admin wants to mint an NFT for a property with ID `101` and an issuance rate of `5%`. The admin would call:

```solidity
manageLife.mint(101, 500);
```

#### Marking a Property as Fully Paid

Once a property with token ID `101` is fully paid, the admin can mark it as such:

```solidity
manageLife.markFullyPaid(16);
```

#### Approving a Third Party

If the homeowner wants to grant access to the marketplace, they would call:

```solidity
manageLife.approve(marketplaceAddress, 7);
```

#### Returning a Property

If the homeowner decides to return the property to ManageLife, they can call:

```solidity
manageLife.returnProperty(1017);
```

### Summary

The ManageLife smart contract leverages the power of blockchain to manage real-world properties and memberships as NFTs. It provides essential features for minting, burning, transferring, and managing these NFTs while integrating with other smart contracts for extended functionalities like token issuance and marketplace interactions.

***

### Function Details

#### `constructor()`

* **Purpose**: Initializes the contract with a name and symbol for the NFT.
* **Rules**: Sets the name "ManageLife Member" and the symbol "MLRE" for the NFT collection.

#### `setBaseURI(string memory newBaseUri)`

* **Purpose**: Updates the base URI for all NFTs.
* **Rules**: Can only be called by the owner. Emits `BaseURIUpdated` event with the new URI.

#### `setMarketplace(address payable marketplace_)`

* **Purpose**: Sets the address of the Marketplace contract.
* **Rules**: Can only be called by the owner.

#### `setLifeToken(address lifeToken_)`

* **Purpose**: Sets the address of the Life token contract.
* **Rules**: Can only be called by the owner.

#### `markFullyPaid(uint256 tokenId)`

* **Purpose**: Marks a property as fully paid and initiates staking rewards.
* **Rules**: Can only be called by the owner. Emits `FullyPaid` event. If the property is not owned by the admin, it initializes staking rewards.

#### `mint(uint256 propertyId, uint256 lifeTokenIssuanceRate_)`

* **Purpose**: Mints a new NFT representing a property.
* **Rules**: Can only be called by the owner. Requires that the Life token contract is set and the property ID has not been minted yet. Sets the issuance rate for the property.

#### `burn(uint256 tokenId)`

* **Purpose**: Burns an NFT, removing it from existence.
* **Rules**: Can only be called by the owner. Uses the `_burn` function from ERC721 and ERC721URIStorage.

#### `retract(uint256 tokenId)`

* **Purpose**: Transfers a property back to the admin.
* **Rules**: Can only be called by the owner. Uses `_safeTransfer` to move the property to the admin.

#### `returnProperty(uint256 tokenId)`

* **Purpose**: Allows a property owner to return the property to the admin.
* **Rules**: Can only be called by the property owner. Uses `safeTransferFrom` to move the property to the admin.

#### `approve(address to, uint256 tokenId)`

* **Purpose**: Approves a third party to manage a property.
* **Rules**: Allows approval if the property is fully paid, owned by the admin, or if the third party is the marketplace. Overrides the `approve` function in ERC721.

#### `_beforeTokenTransfer(address from, address to, uint256 tokenId)`

* **Purpose**: Enforces rules before transferring an NFT.
* **Rules**: Ensures that the property is fully paid or involved parties include the admin or marketplace. Initializes or claims staking rewards based on the transfer. Emits `StakingInitialized` event. Overrides `_beforeTokenTransfer` in ERC721.

#### `updateLifeTokenIssuanceRate(uint256 tokenId, uint256 newLifeTokenIssuanceRate)`

* **Purpose**: Updates the token issuance rate for a property.
* **Rules**: Can only be called by the owner. Claims staking rewards before updating the issuance rate. Updates the start of staking and emits `TokenIssuanceRateUpdated` event.

## Summary

The ManageLife Member's smart contract leverages the power of blockchain to manage real-world properties and memberships as NFTs. It provides essential features for minting, burning, transferring, and managing these NFTs while integrating with other smart contracts for extended functionalities like token issuance and marketplace interactions.
