# ERC721p - Protecc

An update to implement backup / cold wallet addresses to recover NFTs using specified backup addresses.
This is a simple implementation for a specified address to be able to transfer an NFT without actually holding or owning the NFT. This allows for users who have been hacked, phished or otherwise scammed / lost their NFT / lost access to their wallet to be able to get their assets back using a secure backup address.


Note marketplaces like OpenSea will need to be aware for this standard and use superTransfer / superApprove, as otherwise the seller could just take the NFT back after a sale.


**Not audited. Not reviewed. Accept no liability use at own risk.**

## Abstract
An update to implement backup / cold wallet addresses to recover NFTs using specified backup addresses.
This is a simple implementation for a specified address to be able to transfer an NFT without actually holding or owning the NFT. This allows for users who have been hacked, phished or otherwise scammed / lost their NFT / lost access to their wallet to be able to get their assets back using a secure backup address. This backup address has ultimate ownership for the NFT without needing to actually hold the NFT allowing the NFT to be used in a hot wallet environment for metaverse, games, community auth and the usual current use cases for NFTs, including staking.


Note marketplaces like OpenSea will need to be aware for this standard and use superTransfer / superApprove, as otherwise the seller could just take the NFT back after a sale. Alternatively they may check that the ```recoveryAddress[tokenID] == address(0)```, as if the address is set to 0x0, the ultimate authority comes down to the actual NFT holder as is currently the case, who then can change the backup address. This can be a simple front end check to not reequire contract upagrades.

## Motivation
The NFT space has been a playground for innovation, with NFTs being used both as purely fun little art pieces and rock pictures as well as high value art, real estate and domains. As the value for these assets increases so do attempts at theft, through compromised contracts, phishing, hacks, as well as users simply losing access to their wallets. As such a standard is hereby proposed that allows for a cold wallet to be specified which may change the actual NFT owner without impacting the use cases for the NFT through coommunity auth, staking, games and any other use cases currently.

## Specification
### Methods

```function changeRecoveryAddress(uint256 tokenId, address newRecoveryAddress)```

Changes the backup cold wallet or backup alternative address that has ultimate control for the NFT. Checks that the msg.sender matches the current backup or if NO backup is set, allows the holder to set the backup, for backwards compatibility with marketplace contracts and allowing compatibility to be created at the front end level or through a helper contract to check that the backup address is not set during a sale, so that the NFT may not be taken back by the seller post sale.


```function recoveryTransfer(uint256 tokenId, address newOwner) external ```
Allows the backup address that's not the NFT holder to transfer the NFT to a new holder address should the NFT be stolen or otherwise lost. The NFT holder does not need to approve or allow this transfer.

```function superTransfer(uint256 tokenId, address newOwner) external```
Allows the backup address or a Superapproved addressed to transfer the backup address as well as the NFT to the new address, setting the new holder as both the NFT holder (owner) as well as the backup address for the NFT. This is used when ultimate ownership transfer is desired like during sales or transfer to new owner. Can be used interchangably as a replacement for _transferFrom with super authorisation.

```function superApprove(uint256 tokenId, address to) external```
Allows the backup address to Superapprove a address to transfer the backup address as well as the NFT to athe new address, setting the new holder as both the NFT holder (owner) as well as the backup address for the NFT. This is used when ultimate ownership transfer is desired like during sales or transfer to new owners. 

## Rationale
The holder and the backup address have been seperated to allow hot wallets to hold the NFTs and use them for auths for communities, games, staking, as needed while also allowing for the NFT to be returned to the owner if a hack, phishing attack or private key loss happens. The ability for the NFT holder to set the backup address when an 0x0 address (no backup) is set, allows for marketplaces to be backward compatible with the change and only need a helper contract for hard guarantees or a simple front end update for soft guarantees (as a miner may change backup and take back ownership after detecting a mempool update to buy the NFT from a marketplace without a helper intermediate contracct reverting if the backup address != 0x0).

For all other purposes as long as pepople are aware that they need to check and change the backup address when ultimate ownership transfer is desired then all other use cases should not be impacted by the new safety features.  

## Backwards Compatibility
Note marketplaces like OpenSea will need to be aware for this standard and use superTransfer / superApprove, as otherwise the seller could just take the NFT back after a sale. AlLternatively they may check that the ```recoveryAddress[tokenID] == address(0)```, as if the address is set to 0x0, the ultimate authority comes down to the actual NFT holder as is currently the case, who then can change the backup address. This can be a simple front end check to not reequire contract upagrades.

For all other purposes as long as pepople are aware that they need to check and change the backup address when ultimate ownership transfer is desired then all other use cases should not be impacted by the new safety features.  

Staking contracts and contracts wheere actual value is involved may want to check that the NFT has not left the contract before paying out rewards as they may be able to be attacked by making the assumption that they have ultimate control for the NFTs, as the backup can transfer ownership out after staking. 

## Reference Implementation

```
// SPDX-License-Identifier: CC0

pragma solidity ^0.8.0;

import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/utils/Context.sol";
import "@openzeppelin/contracts/utils/Strings.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Metadata.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/IERC721Metadata.sol";
import "@openzeppelin/contracts/utils/introspection/ERC165.sol";

/**
 * @dev Implementation of https://eips.ethereum.org/EIPS/eip-721[ERC721] Non-Fungible Token Standard, including
 * the Metadata extension, but not including the Enumerable extension, which is available separately as
 * {ERC721Enumerable}. With new Protecc for backup addresses.
 */
contract ERC721 is Context, ERC165, IERC721, IERC721Metadata{
    using Address for address;
    using Strings for uint256;

    // Token name
    string private _name;

    // Token symbol
    string private _symbol;

    // Mapping from token ID to owner address
    mapping(uint256 => address) private _owners;

    // Mapping owner address to token count
    mapping(address => uint256) private _balances;

    // Mapping from token ID to approved address
    mapping(uint256 => address) private _tokenApprovals;

    // Mapping for backup address
    mapping(uint256 => address) private recoveryAddress;
    mapping(uint256 => address) public tokenSuperApprovals;

    // Mapping from owner to operator approvals
    mapping(address => mapping(address => bool)) private _operatorApprovals;

    //Update for backup address
    event UpdateRecoveryAddress(uint256, address);

    /**
     * @dev Initializes the contract by setting a `name` and a `symbol` to the token collection.
     */
    constructor(string memory name_, string memory symbol_) {
        _name = name_;
        _symbol = symbol_;
    }

    /**
     * @dev See {IERC165-supportsInterface}.
     */
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(ERC165, IERC165)
        returns (bool)
    {
        return
            interfaceId == type(IERC721).interfaceId ||
            interfaceId == type(IERC721Metadata).interfaceId ||
            super.supportsInterface(interfaceId);
    }

    /**
     * @dev See {IERC721-balanceOf}.
     */
    function balanceOf(address owner)
        public
        view
        virtual
        override
        returns (uint256)
    {
        require(
            owner != address(0),
            "ERC721: balance query for the zero address"
        );
        return _balances[owner];
    }

    /**
     * @dev See {IERC721-ownerOf}.
     */
    function ownerOf(uint256 tokenId)
        public
        view
        virtual
        override
        returns (address)
    {
        address owner = _owners[tokenId];
        require(
            owner != address(0),
            "ERC721: owner query for nonexistent token"
        );
        return owner;
    }

    /**
     * @dev See {IERC721Metadata-name}.
     */
    function name() public view virtual override returns (string memory) {
        return _name;
    }

    /**
     * @dev See {IERC721Metadata-symbol}.
     */
    function symbol() public view virtual override returns (string memory) {
        return _symbol;
    }

    /**
     * @dev See {IERC721Metadata-tokenURI}.
     */
    function tokenURI(uint256 tokenId)
        public
        view
        virtual
        override
        returns (string memory)
    {
        require(
            _exists(tokenId),
            "ERC721Metadata: URI query for nonexistent token"
        );

        string memory baseURI = _baseURI();
        return
            bytes(baseURI).length > 0
                ? string(abi.encodePacked(baseURI, tokenId.toString()))
                : "";
    }

    /**
     * @dev Base URI for computing {tokenURI}. If set, the resulting URI for each
     * token will be the concatenation of the `baseURI` and the `tokenId`. Empty
     * by default, can be overriden in child contracts.
     */
    function _baseURI() internal view virtual returns (string memory) {
        return "";
    }

    /**
     * @dev See {IERC721-approve}.
     */
    function approve(address to, uint256 tokenId) public virtual override {
        address owner = ERC721.ownerOf(tokenId);
        require(to != owner, "ERC721: approval to current owner");

        require(
            _msgSender() == owner || isApprovedForAll(owner, _msgSender()),
            "ERC721: approve caller is not owner nor approved for all"
        );

        _approve(to, tokenId);
    }

    /**
     * @dev See {IERC721-getApproved}.
     */
    function getApproved(uint256 tokenId)
        public
        view
        virtual
        override
        returns (address)
    {
        require(
            _exists(tokenId),
            "ERC721: approved query for nonexistent token"
        );

        return _tokenApprovals[tokenId];
    }

    /**
     * @dev See {IERC721-setApprovalForAll}.
     */
    function setApprovalForAll(address operator, bool approved)
        public
        virtual
        override
    {
        require(operator != _msgSender(), "ERC721: approve to caller");

        _operatorApprovals[_msgSender()][operator] = approved;
        emit ApprovalForAll(_msgSender(), operator, approved);
    }

    /**
     * @dev See {IERC721-isApprovedForAll}.
     */
    function isApprovedForAll(address owner, address operator)
        public
        view
        virtual
        override
        returns (bool)
    {
        return _operatorApprovals[owner][operator];
    }

    /**
     * @dev See {IERC721-transferFrom}.
     */
    function transferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public virtual override {
        //solhint-disable-next-line max-line-length
        require(
            _isApprovedOrOwner(_msgSender(), tokenId),
            "ERC721: transfer caller is not owner nor approved"
        );

        _transfer(from, to, tokenId);
    }

    /**
     * @dev See {IERC721-safeTransferFrom}.
     */
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId
    ) public virtual override {
        safeTransferFrom(from, to, tokenId, "");
    }

    /**
     * @dev See {IERC721-safeTransferFrom}.
     */
    function safeTransferFrom(
        address from,
        address to,
        uint256 tokenId,
        bytes memory _data
    ) public virtual override {
        require(
            _isApprovedOrOwner(_msgSender(), tokenId),
            "ERC721: transfer caller is not owner nor approved"
        );
        _safeTransfer(from, to, tokenId, _data);
    }

    /**
     * @dev Safely transfers `tokenId` token from `from` to `to`, checking first that contract recipients
     * are aware of the ERC721 protocol to prevent tokens from being forever locked.
     *
     * `_data` is additional data, it has no specified format and it is sent in call to `to`.
     *
     * This internal function is equivalent to {safeTransferFrom}, and can be used to e.g.
     * implement alternative mechanisms to perform token transfer, such as signature-based.
     *
     * Requirements:
     *
     * - `from` cannot be the zero address.
     * - `to` cannot be the zero address.
     * - `tokenId` token must exist and be owned by `from`.
     * - If `to` refers to a smart contract, it must implement {IERC721Receiver-onERC721Received}, which is called upon a safe transfer.
     *
     * Emits a {Transfer} event.
     */
    function _safeTransfer(
        address from,
        address to,
        uint256 tokenId,
        bytes memory _data
    ) internal virtual {
        _transfer(from, to, tokenId);
        require(
            _checkOnERC721Received(from, to, tokenId, _data),
            "ERC721: transfer to non ERC721Receiver implementer"
        );
    }

    /**
     * @dev Returns whether `tokenId` exists.
     *
     * Tokens can be managed by their owner or approved accounts via {approve} or {setApprovalForAll}.
     *
     * Tokens start existing when they are minted (`_mint`),
     * and stop existing when they are burned (`_burn`).
     */
    function _exists(uint256 tokenId) internal view virtual returns (bool) {
        return _owners[tokenId] != address(0);
    }

    /**
     * @dev Returns whether `spender` is allowed to manage `tokenId`.
     *
     * Requirements:
     *
     * - `tokenId` must exist.
     */
    function _isApprovedOrOwner(address spender, uint256 tokenId)
        internal
        view
        virtual
        returns (bool)
    {
        require(
            _exists(tokenId),
            "ERC721: operator query for nonexistent token"
        );
        address owner = ERC721.ownerOf(tokenId);
        return (spender == owner ||
            getApproved(tokenId) == spender ||
            isApprovedForAll(owner, spender));
    }

    /**
     * @dev Safely mints `tokenId` and transfers it to `to`.
     *
     * Requirements:
     *
     * - `tokenId` must not exist.
     * - If `to` refers to a smart contract, it must implement {IERC721Receiver-onERC721Received}, which is called upon a safe transfer.
     *
     * Emits a {Transfer} event.
     */
    function _safeMint(address to, uint256 tokenId) internal virtual {
        _safeMint(to, tokenId, "");
    }

    /**
     * @dev Same as {xref-ERC721-_safeMint-address-uint256-}[`_safeMint`], with an additional `data` parameter which is
     * forwarded in {IERC721Receiver-onERC721Received} to contract recipients.
     */
    function _safeMint(
        address to,
        uint256 tokenId,
        bytes memory _data
    ) internal virtual {
        _mint(to, tokenId);
        require(
            _checkOnERC721Received(address(0), to, tokenId, _data),
            "ERC721: transfer to non ERC721Receiver implementer"
        );
    }

    /**
     * @dev Mints `tokenId` and transfers it to `to`.
     *
     * WARNING: Usage of this method is discouraged, use {_safeMint} whenever possible
     *
     * Requirements:
     *
     * - `tokenId` must not exist.
     * - `to` cannot be the zero address.
     *
     * Emits a {Transfer} event.
     */
    function _mint(address to, uint256 tokenId) internal virtual {
        require(to != address(0), "ERC721: mint to the zero address");
        require(!_exists(tokenId), "ERC721: token already minted");

        _beforeTokenTransfer(address(0), to, tokenId);

        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(address(0), to, tokenId);
    }

    /**
     * @dev Destroys `tokenId`.
     * The approval is cleared when the token is burned.
     *
     * Requirements:
     *
     * - `tokenId` must exist.
     *
     * Emits a {Transfer} event.
     */
    function _burn(uint256 tokenId) internal virtual {
        address owner = ERC721.ownerOf(tokenId);

        _beforeTokenTransfer(owner, address(0), tokenId);

        // Clear approvals
        _approve(address(0), tokenId);

        _balances[owner] -= 1;
        delete _owners[tokenId];

        emit Transfer(owner, address(0), tokenId);
    }

    /**
     * @dev Transfers `tokenId` from `from` to `to`.
     *  As opposed to {transferFrom}, this imposes no restrictions on msg.sender.
     *
     * Requirements:
     *
     * - `to` cannot be the zero address.
     * - `tokenId` token must be owned by `from`.
     *
     * Emits a {Transfer} event.
     */
    function _transfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {
        require(
            ERC721.ownerOf(tokenId) == from,
            "ERC721: transfer of token that is not own"
        );
        require(to != address(0), "ERC721: transfer to the zero address");

        _beforeTokenTransfer(from, to, tokenId);

        // Clear approvals from the previous owner
        _approve(address(0), tokenId);

        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;

        emit Transfer(from, to, tokenId);
    }

    /**
     * @dev Approve `to` to operate on `tokenId`
     *
     * Emits a {Approval} event.
     */
    function _approve(address to, uint256 tokenId) internal virtual {
        _tokenApprovals[tokenId] = to;
        emit Approval(ERC721.ownerOf(tokenId), to, tokenId);
    }

    //-------------------------------------Protecc Module------------------------

    function changeRecoveryAddress(uint256 tokenId, address newRecoveryAddress)
        external
    {
        if (recoveryAddress[tokenId] != address(0)) {
            require(
                msg.sender == recoveryAddress[tokenId],
                "not the recovery address"
            );
        } else {
            require(msg.sender == ERC721.ownerOf(tokenId));
        }

        recoveryAddress[tokenId] = newRecoveryAddress;
        _approve(newRecoveryAddress, tokenId);
        emit UpdateRecoveryAddress(tokenId, newRecoveryAddress);
    }

    function recoveryTransfer(uint256 tokenId, address newOwner) external {
        require(
            msg.sender == recoveryAddress[tokenId],
            "not the recovery address"
        );
        _safeTransfer(ownerOf(tokenId), newOwner, tokenId, "");
    }

    function superTransfer(uint256 tokenId, address newOwner) external {
        require(
            msg.sender == recoveryAddress[tokenId] ||
                msg.sender == tokenSuperApprovals[tokenId],
            "not the recovery address"
        );
        recoveryAddress[tokenId] = newOwner;
        _safeTransfer(ownerOf(tokenId), newOwner, tokenId, "");
    }

    function superApprove(uint256 tokenId, address to) external {
        require(
            msg.sender == recoveryAddress[tokenId],
            "not the recovery address"
        );
        tokenSuperApprovals[tokenId] = to;
    }

    // -----------------------------------------------------------------------------
    /**
     * @dev Internal function to invoke {IERC721Receiver-onERC721Received} on a target address.
     * The call is not executed if the target address is not a contract.
     *
     * @param from address representing the previous owner of the given token ID
     * @param to target address that will receive the tokens
     * @param tokenId uint256 ID of the token to be transferred
     * @param _data bytes optional data to send along with the call
     * @return bool whether the call correctly returned the expected magic value
     */
    function _checkOnERC721Received(
        address from,
        address to,
        uint256 tokenId,
        bytes memory _data
    ) private returns (bool) {
        if (to.isContract()) {
            try
                IERC721Receiver(to).onERC721Received(
                    _msgSender(),
                    from,
                    tokenId,
                    _data
                )
            returns (bytes4 retval) {
                return retval == IERC721Receiver.onERC721Received.selector;
            } catch (bytes memory reason) {
                if (reason.length == 0) {
                    revert(
                        "ERC721: transfer to non ERC721Receiver implementer"
                    );
                } else {
                    assembly {
                        revert(add(32, reason), mload(reason))
                    }
                }
            }
        } else {
            return true;
        }
    }

    /**
     * @dev Hook that is called before any token transfer. This includes minting
     * and burning.
     *
     * Calling conditions:
     *
     * - When `from` and `to` are both non-zero, ``from``'s `tokenId` will be
     * transferred to `to`.
     * - When `from` is zero, `tokenId` will be minted for `to`.
     * - When `to` is zero, ``from``'s `tokenId` will be burned.
     * - `from` and `to` are never both zero.
     *
     * To learn more about hooks, head to xref:ROOT:extending-contracts.adoc#using-hooks[Using Hooks].
     */
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 tokenId
    ) internal virtual {}
}
```

## Security Considerations
As noted contracts which assume ultimate ownership transfer such as marketplaces, some staking contracts, actual ownership transfers must be aware that the backup address NOT the NFT holder has ultimate ownership authority. Further the backup address is advised to be a cold wallet address or at the very least a more secure backup alternative adddress than the owner hot wallet that holds the NFT. superApprove should be used with extreme caution only for reputable, audtied and known contracts as if theres a vulnerability the NFT may be stolen as with current systems. 

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
