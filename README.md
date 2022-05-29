# ERC721p - Protecc

### An update to implement backup / cold wallet addresses to recover NFTs using specified backup addresses.
This is a simple implementation for a specified address to be able to transfer an NFT without actually holding or owning the NFT. This allows for users who have been hacked, phished or otherwise scammed / lost their NFT / lost access to their wallet to be able to get their assets back using a secure backup address.


Note marketplaces like OpenSea will need to be aware for this standard and use superTransfer / superApprove, as otherwise the seller could just take the NFT back after a sale.


Not audited. Not reviewed. Accept no liability use at own risk.
